---
title: Простой SNMP сервер на C#
date: 2021-08-12 15:22:45
tags: [C#, SNMP, server]
---

Простой SNMP сервер непрерывно слушает порт (161 для симулятора МФУ) и проверяет присутствие пришедшей по UDP команды OID в словаре dictionary. В качестве ключа dictionary содержит OID команду, а в качестве значения - ответ МФУ на неё. Словарь dictionary формируется при инициализации приложения из appSettings.json. При наличии ключа отправляется ответ отправителю запроса. 

Обратите внимание на метод *string ConvertBytesToStringOid(byte[] oidBytes)*, преобразующий пришедшие байты OID команды в текстовое значение к которому мы привыкли (вида 1.3.6.1.2.1.43.5.1.1.17.1). Именно оно используется для поиска в словаре dictionary.

``` csharp
public static class SnmpServer
{
    public static void StartListening(Dictionary<string, string> dictionary, int port)
    {
        while (true)
        {
            var receiver = new UdpClient(port); // UdpClient для получения данных
            IPEndPoint remoteIp = null; // адрес входящего подключения

            try
            {
                while (true)
                {
                    var data = receiver.Receive(ref remoteIp); // получаем данные
                    int requestIdBytesLength = data[16];
                    var requestIdBytes = new byte[requestIdBytesLength];
                    Buffer.BlockCopy(data, 17, requestIdBytes, 0, requestIdBytesLength);
                    int inputOidLength = data[32];
                    var inputOidBytes = new byte[inputOidLength];
                    Buffer.BlockCopy(data, 33, inputOidBytes, 0, inputOidLength);
                    var inputOid = ConvertBytesToStringOid(inputOidBytes);
                    if (dictionary.TryGetValue(inputOid, out var oidValue))
                    {
                        var sender = new UdpClient(remoteIp.Address.ToString(), remoteIp.Port);
                        var oidValueBytes = Encoding.ASCII.GetBytes(oidValue);
                        var response = GenerateOidBytesResponse(
                            requestIdBytes, 
                            inputOidBytes, 
                            oidValueBytes
                        );
                        sender.Send(response, response.Length); // отправка
                        sender.Close();
                        sender.Dispose();
                    }
                }
            }
            catch (Exception)
            {
                receiver.Close();
            }
        }
    }

    private static byte[] GenerateOidBytesResponse(
        byte[] requestIdBytes, 
        byte[] inputOidBytes, 
        byte[] oidValueBytes
    )
    {
        var result1 = new List<byte>(oidValueBytes);
        result1.Insert(0, (byte)oidValueBytes.Length);
        result1.Insert(0, 0x04);
        var result2 = new List<byte>(inputOidBytes);
        result2.AddRange(result1);
        result2.Insert(0, (byte)inputOidBytes.Length);
        result2.Insert(0, 0x06);
        result2.Insert(0, (byte)result2.Count);
        result2.Insert(0, 0x30);
        result2.Insert(0, (byte)result2.Count);
        result2.Insert(0, 0x30);
        
        Magic210(result2);
        Magic210(result2);

        var result3 = new List<byte>(requestIdBytes);
        result3.AddRange(result2);
        result3.Insert(0, (byte)requestIdBytes.Length);
        result3.Insert(0, 0x02);
        result3.Insert(0, (byte)result3.Count);
        result3.Insert(0, 0xa2);

        CommunityPublic(result3);
        Magic210(result3);
        
        result3.Insert(0, (byte)result3.Count);
        result3.Insert(0, 0x30);

        return result3.ToArray();
    }

    private static void CommunityPublic(List<byte> list)
    {
        list.Insert(0, 0x63);
        list.Insert(0, 0x69);
        list.Insert(0, 0x6c);
        list.Insert(0, 0x62);
        list.Insert(0, 0x75);
        list.Insert(0, 0x70);
        list.Insert(0, 0x06);
        list.Insert(0, 0x04);
    }

    private static void Magic210(List<byte> list)
    {
        list.Insert(0, 0x00);
        list.Insert(0, 0x01);
        list.Insert(0, 0x02);
    }

    private static string ConvertBytesToStringOid(byte[] oidBytes)
    {
        var builder = new StringBuilder();
        
        var result = new List<uint> { (uint)(oidBytes[0] / 40), (uint)(oidBytes[0] % 40) };
        
        uint buffer = 0;
        for (var i = 1; i < oidBytes.Length; i++)
        {
            if ((oidBytes[i] & 0x80) == 0)
            {
                result.Add(oidBytes[i] + (buffer << 7));
                buffer = 0;
            }
            else
            {
                buffer <<= 7;
                buffer += (uint)(oidBytes[i] & 0x7F);
            }
        }

        for (var i = 0; i < result.Count; i++)
        {
            builder.Append(result[i]);
            if (i != result.Count - 1) builder.Append('.');
        }

        return builder.ToString();
    }
}
```