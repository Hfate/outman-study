#Lua脚本
    
     Redis从2.6版本开始引入对Lua脚本的支持，通过在服务器嵌入Lua环境，Redis客户端可以使用Lua脚本，直接在服务器端与乃至的执行U盾讴歌命令
    
   + 创建并修改Lua环境
        + 创建一个基础的Lua环境，之后所有的修改都是针对这个环境进行
        + 载入多个函数库到Lua环境里面，让Lua脚本可以使用这些函数库来进行数据库操作
        + 创建全局表格redis,这个表耳包含了对Redis进行操作的函数，比如用于在Lua脚本中执行Redis命令的redis.call函数
        + 使用Redis自制的随机函数来替换Lua原有的带有副作用的随机函数，从而避免在焦本中引入副作用
        + 创建排序辅助函数，Lua环境使用这个辅佐函数来对一部分Redis命令的结果进行排序，从而消除这些命令的不确定性
        + 创建redis.call函数的错误报告辅助函数，这个函数可以提供更详细的出错信息
        + 对lua环境中全局环境进行保护，防止用户在执行Lua脚本的过程中，将额外的全局变量添加到Lua换中
        + 将完成修改的Lua环境保存到服务器状态的lua属性中，等待执行服务器传来的lua脚本  
        
   + 伪客户端
        redis执行命令必须有客户端状态，故为执行lua脚本创建了一个伪客户端
        + Lua环境将redis.call函数或者redis.pcall函数相应执行的命令传给伪客户端
        + 伪客户端将脚本传给命令执行器
        + 命令执行器执行命令，将执行结果->伪客户端->Lua环境->redis.call函数->调用者
   
   + EVAL命令的实现
        + 根据客户端给定lua脚本，在lua环境中定义一个lua函数
        + 将客户端给地的脚本保存到lua_scripts字典
        + 执行刚刚在lua环境中定义的函数，以此来执行客户单给定的lua脚本
        
        
   + EVALSHA命令的实现
        
         每个eval命令成功执行过的lua脚本，在lua环境里面都有一个与这个脚本相对应的lua函数，函数的名字由f_前缀加上40个字符昌SHA1校验和促成。
         
        ```
          defEVALSHA(sha1):
            #拼接出函数的名字
            func_name = "f_"+sha1
            # 查看这个函数在lua环境中是否存在
            if function_exists_in_lua_env(func_name):
                #如果函数存在，执行它
                execute_lua_function(func_name)
            else
                # 如果函数不存在，那么返回一个错误
                send_script_erroor("SCRIPT NOT FOUND")  
        ```
   + SCRIPT FLUSH
   
         SCRIPT FLUSH命令用于清除服务器中所有和lua脚本有关的信息，这个命令会释放并重建lua_scripts字典，关闭现有的lua环境并重新创建一个新的lua环境
   
   + SCRIPT EXIST
       
         SCRIPT EXIST命令根据输入的SHA1校验和，检查校验和对应的脚本是否存在于服务器中
   
   + SCRIPT LOAD
    
         SCRIPT LOAD所做的事和EVAL命令前两步完全一样。命令会在lua环境中创建对应的函数，然后再将脚本保存到lua_scripts字典里，然后等等EVALSHA调用
         
   + SCRIPT KILL
          
          如果脚本尚未执行写操作，则杀死当前正在执行的 Lua 脚本。
          此命令主要用于杀死运行时间过长的脚本（例如，因为它由于错误而进入无限循环）。该脚本将被终止，并且当前阻止进入 EVAL 的客户端将看到该命令返回错误。
          如果脚本已经执行了写操作，则它不能以这种方式被杀死，因为它会违反 Lua 脚本原子性合约。在这种情况下，只能SHUTDOWN NOSAVE杀死脚本，严重破坏 Redis 进程，阻止它以半写的信息持续存在。
   +       
       
       
              
                  