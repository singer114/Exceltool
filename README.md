# Excel工具箱收费系统

这个项目包含基于GitHub Pages的Excel工具箱收费系统，无需服务器即可实现软件授权验证。

## 文件说明

- `index.html` - 产品主页，介绍软件功能和价格
- `payment.html` - 支付页面，包含微信支付二维码
- `wechat_payment_qr.png` - 微信支付二维码图片
- `licenses.json` - 存储有效授权码的JSON文件
- `README.md` - 项目说明文件

## 部署步骤

1. 创建GitHub仓库并启用GitHub Pages
   - 登录您的GitHub账号
   - 创建一个新的公开仓库(例如：Exceltool)
   - 将这些文件上传到仓库

2. 替换微信收款码图片
   - 将您的微信收款码图像保存为`wechat_payment_qr.png`
   - 上传替换仓库中的同名文件

3. 启用GitHub Pages
   - 进入仓库设置 -> Pages
   - 选择主分支(main)作为发布源
   - 点击保存，等待几分钟后网站将被发布

4. 验证部署
   - 访问`https://singer114.github.io/Exceltool/`确认网站正常运行

## 授权码管理

添加新的授权码：

1. 生成新的MD5哈希值作为授权码
2. 编辑`licenses.json`文件，将新的授权码添加到`keys`数组中
3. 提交更改到GitHub仓库

## 软件授权验证代码

将以下代码添加到main.py文件的开头，以实现授权验证：

```python
import hashlib
import requests
import os
import tkinter as tk
from tkinter import simpledialog, messagebox
import json
import time
from datetime import datetime, timedelta

def load_local_license():
    """从本地文件加载许可证信息"""
    license_path = os.path.join(os.path.dirname(os.path.abspath(__file__)), 'license.dat')
    if os.path.exists(license_path):
        try:
            with open(license_path, 'r') as f:
                data = json.load(f)
                # 验证授权是否过期（如果是临时授权）
                if 'expires' in data:
                    expires = datetime.fromisoformat(data['expires'])
                    if expires < datetime.now():
                        return None
                return data.get('key')
        except:
            return None
    return None

def save_local_license(key, expires=None):
    """保存许可证信息到本地文件"""
    license_path = os.path.join(os.path.dirname(os.path.abspath(__file__)), 'license.dat')
    data = {'key': key}
    if expires:
        data['expires'] = expires.isoformat()
    
    with open(license_path, 'w') as f:
        json.dump(data, f)

def verify_license(license_key):
    """验证授权码是否有效"""
    # 计算授权码哈希
    hash_key = hashlib.md5(license_key.encode()).hexdigest()
    
    # 尝试在线验证
    try:
        # 替换为您的GitHub Pages URL
        response = requests.get("https://singer114.github.io/Exceltool/licenses.json", timeout=5)
        if response.status_code == 200:
            valid_licenses = response.json()
            if hash_key in valid_licenses["keys"]:
                # 有效授权，保存到本地
                save_local_license(hash_key)
                return True
    except:
        # 如果在线验证失败，检查本地缓存的授权
        local_key = load_local_license()
        if local_key and local_key == hash_key:
            return True
            
    # 如果授权码无效，提供试用选项
    return False

def check_license():
    """检查授权状态，如无效则提示用户输入授权码"""
    # 首先检查本地保存的授权信息
    local_key = load_local_license()
    if local_key:
        # 验证本地授权
        try:
            response = requests.get(f"https://singer114.github.io/Exceltool/licenses.json", timeout=5)
            if response.status_code == 200:
                valid_licenses = response.json()
                if local_key in valid_licenses["keys"]:
                    return True
        except:
            # 如果无法联网，允许使用本地授权继续
            return True
    
    # 如果没有有效的授权，显示授权对话框
    root = tk.Tk()
    root.withdraw()  # 隐藏主窗口
    
    messagebox.showinfo("Excel工具箱", "请输入授权码激活软件\n如需购买授权码，请访问：https://singer114.github.io/Exceltool/")
    
    # 初始化试用状态
    trial_started = False
    max_attempts = 3
    attempts = 0
    
    while attempts < max_attempts:
        license_key = simpledialog.askstring("授权验证", "请输入授权码：\n(点击'取消'可选择试用版)", parent=root)
        
        # 用户取消输入，提供试用选项
        if license_key is None:
            trial_response = messagebox.askyesno("试用版", "是否启动15天试用版？")
            if trial_response:
                # 创建临时授权
                temp_key = f"trial-{int(time.time())}"
                expires = datetime.now() + timedelta(days=15)
                save_local_license(temp_key, expires)
                messagebox.showinfo("试用已激活", f"您的试用版将于 {expires.strftime('%Y-%m-%d')} 到期")
                return True
            else:
                # 用户不想试用，退出应用
                messagebox.showinfo("感谢使用", "感谢您对Excel工具箱的关注！")
                root.destroy()
                sys.exit(0)
        
        # 验证用户输入的授权码
        if verify_license(license_key):
            messagebox.showinfo("激活成功", "Excel工具箱已成功激活！")
            root.destroy()
            return True
        else:
            attempts += 1
            remaining = max_attempts - attempts
            if remaining > 0:
                messagebox.showerror("授权失败", f"授权码无效，请重新输入。\n还剩 {remaining} 次尝试机会。")
            else:
                messagebox.showerror("授权失败", "多次输入错误，程序将退出。")
                root.destroy()
                sys.exit(0)
    
    root.destroy()
    return False

# 程序启动时检查授权
if __name__ == "__main__":
    if not check_license():
        sys.exit(0)
    
    # 继续执行原有程序...
```

修改主程序的启动部分，添加授权检查：

```python
if __name__ == "__main__":
    # 检查授权
    if not check_license():
        sys.exit(0)
        
    app = QApplication(sys.argv)
    window = FunctionSelectionWindow()
    window.show()
    sys.exit(app.exec_())
``` 