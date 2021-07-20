---
title: Введение в синтаксис NSIS скриптов
date: 2021-07-18 13:02:25
tags: [NSIS, nsis, script, scripting]
---

### Типичный скрипт NSIS состоит из:
* Глобальных переменных (Var nameServer)
* Стэка
* Регистров ($0-$9,$R0-$R9)
* Встроенных функций (StrCmp, IntCmp, IfErrors, Goto ...)
``` nsis
StrCmp $0 'some value' 0 +3
  MessageBox MB_OK '$$0 is some value'
  Goto done
StrCmp $0 'some other value' 0 +3
  MessageBox MB_OK '$$0 is some other value'
  Goto done
# else
  MessageBox MB_OK '$$0 is "$0"'
done:
```
* удобной библиотеки LogicLib, превращающей логические операции встроенных функций в некое подобие высокоуровневой библиотеки
``` nsis
${If} $0 == 'some value'
  MessageBox MB_OK '$$0 is some value'
${ElseIf} $0 == 'some other value'
  MessageBox MB_OK '$$0 is some other value'
${Else}
  MessageBox MB_OK '$$0 is "$0"'
${EndIf}

${Switch}, ${If}, ${While}, ${For} etc.
```
* Comments  ; # /**/ 
* Next line hyphenation   \
* Var BLA ;Declare the variable
* StrCpy $BLA "123" ;Now you can use the variable $BLA
* !include LogicLib.nsh
* !define APPNAME "Installer“

* Branching
``` nsis
${If} $Dialog == error
	Abort
${EndIf}

${IfNot} ${Silent}
${AndIf} ${SectionIsSelected} ${SectionAgent}
	...code...
${EndIf}
```

* Functions
``` nsis
Function bla
  Push $R0
  Push $R1
    ...code...
  Pop $R1
  Pop $R0
FunctionEnd

Call bla  ;call the function. Arguments pass by stack

Function un.bla
  ...code...
FunctionEnd
```

* Macros
``` nsis
!macro LogDetailPrintMessageBox message
	!insertmacro GetTimeStampString $R9
	nsislog::log "$EXEDIR\${LOGFILE}" "$R9: ${message}"
	${IfNot} ${Silent}
		DetailPrint "${message}"
		MessageBox MB_OK "${message}"
	${EndIf}
!macroend
```

* MessageBoxes
MessageBox MB_OK "simple message box"
MessageBox MB_YESNO "is it true?" IDYES true IDNO false
true:
  DetailPrint "it's true!"
  Goto next
false:
  DetailPrint "it's false"
next:

* DetailPrint
DetailPrint "this message will show on the installation window"

* Sections

Section "-hidden section"
SectionEnd

Section # hidden section
SectionEnd

Section "!bold section"
SectionEnd

Section /o "optional"
SectionEnd

Section "install something" SEC_IDX
SectionEnd

InstType "full"
InstType "minimal"

Section "a section"
    SectionIn 1 2
SectionEnd

Section "another section"
     SectionIn 1
SectionEnd

Section "Uninstall"
  Delete $INSTDIR\Uninst.exe ; delete self (see explanation below why this works)
  Delete $INSTDIR\myApp.exe
  RMDir $INSTDIR
  DeleteRegKey HKLM SOFTWARE\myApp
SectionEnd

* Plug-in DLLs

SimpleSC::ExistsService "$paramServiceName"
Pop $0 ; returns an errorcode if the service doesn?t exists (<>0)/service exists (0)
${If} $0 == 0
	Call smartRemover
	Abort
${EndIf}

ClearErrors
nsJSON::Serialize /format /file "$INSTDIR\${appName}\appsettings.json" ;save to file
${If} ${Errors}
	Call smartRemover
	Abort
${EndIf}

* Modern User Interface

Installer pages
MUI_PAGE_WELCOME
MUI_PAGE_LICENSE textfile
MUI_PAGE_COMPONENTS
MUI_PAGE_DIRECTORY
MUI_PAGE_STARTMENU pageid variable
MUI_PAGE_INSTFILES
MUI_PAGE_FINISH

Uninstaller pages
MUI_UNPAGE_WELCOME
MUI_UNPAGE_CONFIRM
MUI_UNPAGE_LICENSE textfile
MUI_UNPAGE_COMPONENTS
MUI_UNPAGE_DIRECTORY
MUI_UNPAGE_INSTFILES
MUI_UNPAGE_FINISH
