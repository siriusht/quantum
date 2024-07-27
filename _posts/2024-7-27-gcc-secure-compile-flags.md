| 安全编译选项      | 描述                   | 级别   | 编译命令                                                     | 验证                                                         |
| ----------------- | ---------------------- | ------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| BIND_NOW          | 立即绑定               | high   | -Wl,-z,now                                                   | readelf -d                                                   |
| NX                | 堆栈不可执行           | high   | -Wl,-z,noexecstack                                           | readelf -l GNU_STACK                                         |
| PIC               | 地址无关               | high   | -fPIC, for shared library                                    |                                                              |
| PIE               | 随机化                 | high   | -fPIE, for executables                                       |                                                              |
| RELRO             | GOT表保护              | high   | -Wl,-z,relro,-z,now                                          | checksec                                                     |
| SP                | 栈保护                 | high   | -fstack-protector-all/-fstack-protector-strong               |                                                              |
| NO  Rpath/Runpath | 动态库搜索路径（禁选） | high   | -Wl,--disable-new-dtags <br />cmake: set(CMAKE_SKIP_BUILD_RPATH TRUE) | readelf -d                                                   |
| FS                | Fortify Source         | medium | -D_FORTIFY_SOURCE=2                                          |                                                    |
| Ftrapv            | 整数溢出检查           | medium | -ftrapv                                                      | `-ftrapv` does not work in GCC, https://gcc.gnu.org/bugzilla/show_bug.cgi?id=35412有bug，待商榷 |
| Strip             | 删除符号表             | high   | -s                                                           | checksec                                                     |
