---
date: '2026-04-30T10:24:27+08:00'
draft: false
title: 'Browser Harness'
---

"Don't wrap the LLM. Don't wrap its tools either."



# AI接管你的浏览器

如果你希望agent可以直接帮你操作你的浏览器，例如重复性网页操作，网页下载收集资料等等，Browser Harness是一个值得关注的项目。相比于他们之前的项目Browser-use框架，前者是一个完整 Web Agent 框架，它会提供大量已经封装好的能力，Browser Harness是一个极小的浏览器 harness，给 Agent 一个真实浏览器，剩下的全都交给它自己。一个简单的例子，假设任务是`打开一个网站，搜索某个关键词，提取前 5 条结果。`Browser-use会根据模型选择动作，选择 click / input / scroll 等 action，执行动作，再次观察，直到完成任务。但遇到框架边界之外的的任务，就需要人为打补丁，那不如把权限都交给agent本身，Browser Harness就出现了。它会自己写 helper、调试错误、调用 API、处理文件、保存成skill等等操作。两者最大的区别是把承担构建任务框架的角色交给agent还是程序员。

Browser Harness不需要编排复杂框架，只给他基础的CDP（Chrome DevTools Protocol），让Agent自己在浏览器探索完成任务，发现规律、踩坑、写 helper，经验保存成skills。

![图片说明](/images/browseharness1.png)

# 四个文件
## run.py

Browser Harness 使用方式非常暴力，SKILL.md 明确说：调用方式就是 browser-harness -c '任意 Python'，helpers 会预导入，daemon 会自动启动，不需要手动管理。
```bash
browser-harness -c '
new_tab("https://docs.browser-use.com")
wait_for_load()
print(page_info())
'
```

通过源码可以看到，run.py 主要做几件事：设置 Windows 输出编码、导入管理函数和 helpers、解析命令行参数、确保 daemon 启动，然后执行 -c 后面传入的 Python 代码。
```python
def main():
    args = sys.argv[1:]
    if args and args[0] in {"-h", "--help"}:
        print(HELP)
        return
    if args and args[0] == "--version":
        print(_version() or "unknown")
        return
    if args and args[0] == "--doctor":
        sys.exit(run_doctor())
    if args and args[0] == "--setup":
        sys.exit(run_setup())
    if args and args[0] == "--update":
        yes = any(a in {"-y", "--yes"} for a in args[1:])
        sys.exit(run_update(yes=yes))
    if args and args[0] == "--reload":
        restart_daemon()
        print("daemon stopped — will restart fresh on next call")
        return
    if args and args[0] == "--debug-clicks":
        os.environ["BH_DEBUG_CLICKS"] = "1"
        args = args[1:]
    if not args or args[0] != "-c":
        sys.exit("Usage: browser-harness -c \"print(page_info())\"")
    if len(args) < 2:
        sys.exit("Usage: browser-harness -c \"print(page_info())\"")
    print_update_banner()
    ensure_daemon()
    exec(args[1], globals())
```
browser-harness支持一些指令：
```
--help      显示帮助
--version   显示版本
--doctor    检查环境
--setup     初始化配置
--update    更新工具
--reload    重启 daemon
```

如果用户传入的是`browser-harness -c "XXXX"`, 最后到run.py最核心的地方三行，`print_update_banner()`检查/提示是否需要更新 browser-harness，`ensure_daemon()`确保后台 daemon 已经启动，并且已经连接到 Chrome CDP，`exec(args[1], globals())`把 -c 后面的字符串当作 Python 代码执行。