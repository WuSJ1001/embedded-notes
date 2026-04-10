​博客教程：
[office全家桶安装与激活 - Ryougi9 - 博客园](https://www.cnblogs.com/ryougi9/p/18692148)
官方网站：
[massgravel/Microsoft-Activation-Scripts: Open-source Windows and Office activator featuring HWID, Ohook, TSforge, and Online KMS activation methods, along with advanced troubleshooting.](https://github.com/massgravel/Microsoft-Activation-Scripts)

搜索PowerShell，右键管理员方式打开  
输入 
```
irm https://get.activated.win | iex
```
等待启动，完成将会弹出新的窗口


# 说明
### 激活方法 (Activation Methods)

- **[1] HWID - Windows**
    
    - **翻译：** 硬件 ID 激活。
        
    - **解释：** **最推荐**的 Windows 激活方式。它是永久激活，会将你的电脑主板信息注册到微软服务器，重装系统后只要联网就会自动激活。
        
- **[2] Ohook - Office**
    
    - **翻译：** Ohook 激活。
        
    - **解释：** **最推荐**的 Office 激活方式。它是永久激活，不修改系统文件，支持全系列 Office 软件。
        
- **[3] TSforge - Windows / Office / ESU**
    
    - **解释：** 另一种激活技术，主要用于上面提到的 ESU 更新或特定版本的激活。
        
- **[4] Online KMS - Windows / Office**
    
    - **翻译：** 在线 KMS 激活。
        
    - **解释：** 传统的激活方式。通常有效期为 180 天（该脚本会自动创建续期任务使其看起来像永久），适用于批量授权版本。
        

---

### 管理与工具

- **[5] Check Activation Status:** 检查激活状态（查看你现在的 Windows/Office 是否已激活）。
    
- **[6] Change Windows Edition:** 更改 Windows 版本（例如从“家庭版”切换到“专业版”）。
    
- **[7] Change Office Edition:** 更改 Office 版本。
    
- **[8] Troubleshoot:** 故障排除（解决激活失败的问题）。
    
- **[E] Extras:** 额外功能（一些高级设置）。
    
- **[H] Help:** 帮助。
    
- **[0] Exit:** 退出。