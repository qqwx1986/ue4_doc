加载顺序问题，只需要在相应的.uplugin 中的相应的 Module 中 LoadingPhase 设置为EarliestPossible既可以

注册表
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager

DWORD DebuggerMaxModuleMsgs = 2048
