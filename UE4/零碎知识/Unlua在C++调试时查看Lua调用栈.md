1. 在调试面板定位Lua调用C++的栈帧，一般位于LuaCore.cpp这个文件上
2. 右侧Evaluate expression的输入框填入`UnLua::PrintCallStack(L)`，按回车
3. 切换到Debugger Output(LLDB)面板，就可以看到lua侧的调用栈了