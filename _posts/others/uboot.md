# uboot



## make xxx_defconfig



### Top Makefile : %config

- 依赖

  - scripts_basic

    - $(Q)$(MAKE) $(build)=scripts/basic

      > $(build) 定义在 scripts/Kbuild.include中  
      >
      > build := -f $(srctree)/scripts/Makefile.build obj
      >
      > 即 命令展开为 @ make -f  $(srctree)/scripts/Makefile.build obj=scripts/basic
      >
      > 编译生成了 scripts/basic/fixdep

  -  outputmakefile 
  - FORCE 

- 命令 

  -  $(Q)$(MAKE) $(build)=scripts/kconfig $@ 
    展开为 @ make -f  $(srctree)/scripts/Makefile.build obj=scripts/kconfig   xxx_defconfig

    > src = scripts/kconf
    >
    > kbuild-dir = scripts/kconf
    >
    > kbuild-file = scripts/kconf/Makefile
    >
    > include scripts/kconf/Makefile    # xxx_defconfig 目标定义在这个文件中
    >
    > 
    >
    > %_defconfig: $(obj)/conf                                                        
    >         $(Q)$< $(silent) --defconfig=arch/$(SRCARCH)/configs/$@ $(Kconfig)
    >
    > $<表示第一个依赖文件， $^ 表示所有依赖文件， $@ 表示目标
    >
    > 即展开为 @scripts/kconfig/conf  -s  --defconfig=arch/$(SRCARCH)/configs/xxx_defconfig  $(Kconfig)
    >
    > 通过[对conf源码的分析](https://blog.csdn.net/guyongqiangx/article/details/52679635)，大概知道conf程序用于维护 .config ，
    >
    > - 如果传入了--defconfig参数就读取 Kconfig文件生成 .config文件
    > - 如果传入了 --silentoldconfig （在make 整个u-boot时）则生成以下文件
    >   - `include/generated/autoconf.h`   是C语言头文件，主要影响C语言编译过程。
    >   - `include/config/auto.conf.cmd`  
    >   -  `include/config/tristate.conf`  
    >   -  `include/config/auto.conf`   被顶层Makefile包含，影响编译过程。

    

## make

![uboot-make](/images/u-boot/uboot-make.png)

