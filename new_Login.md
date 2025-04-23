# Django 跨站点自动登录方案

这个项目实现了两个 Django 网站之间的自动登录功能。当用户在 artdott.com 网站登录后，可以通过点击按钮自动跳转并登录到 curiosityroom.xrexp.io 网站。该方案使用 JWT (JSON Web Token) 实现安全的跨站点身份验证。

## 功能特点

- 基于 JWT 的安全身份验证
- 自动用户信息同步
- 简单易用的接口
- 支持 HTTPS
- 跨域请求支持

## 技术栈

- Python 3.8+
- Django 3.2+
- PyJWT 2.6.0
- django-cors-headers 3.14.0

## 安装步骤

1. 克隆项目到本地

```bash
git clone <your-repository-url>
cd <project-directory>
```

2. 创建并激活虚拟环境

```bash
python -m venv venv
source venv/bin/activate  # Linux/Mac
# 或
.\venv\Scripts\activate  # Windows
```

3. 安装依赖

```bash
pip install -r requirements.txt
```

4. 配置环境变量

在两个项目的根目录下分别创建 `.env` 文件：

```env
SECRET_KEY=your-shared-secret-key
DEBUG=False
ALLOWED_HOSTS=your-domain.com
```

## 配置说明

### artdott.com 网站（源站点）配置

1. 添加视图代码到 `views.py`:

```python
from django.shortcuts import redirect
import jwt
import time

def redirect_to_target_site(request):
    if not request.user.is_authenticated:
        return redirect('login')
    
    payload = {
        'user_id': request.user.id,
        'username': request.user.username,
        'email': request.user.email,
        'exp': int(time.time()) + 300,
        'iat': int(time.time())
    }
    
    SECRET_KEY = 'your-shared-secret-key'
    token = jwt.encode(payload, SECRET_KEY, algorithm='HS256')
    
    target_site_url = 'https://curiosityroom.xrexp.io/user/sso/login/'
    redirect_url = f'{target_site_url}?token={token}'
    
    return redirect(redirect_url)
```

2. 更新 `urls.py`:

```python
from django.urls import path
from . import views

urlpatterns = [
    path('redirect-to-b/', views.redirect_to_target_site, name='redirect_to_b'),
]
```

### curiosityroom.xrexp.io 网站（目标站点）配置

1. 添加视图代码到 `views.py`:

```python
from django.shortcuts import redirect
from django.contrib.auth import login
from django.contrib.auth.models import User
import jwt
from django.conf import settings

class SSOLoginView(View):
    def get(self, request):
        token = request.GET.get('token')
        if not token:
            return JsonResponse({'error': 'No token provided'}, status=400)
            
        try:
            SECRET_KEY = 'your-shared-secret-key'
            payload = jwt.decode(token, SECRET_KEY, algorithms=['HS256'])
            
            user, created = User.objects.get_or_create(
                username=payload['username'],
                defaults={
                    'email': payload['email'],
                }
            )
            
            login(request, user)
            return redirect('home')
            
        except jwt.ExpiredSignatureError:
            return JsonResponse({'error': 'Token has expired'}, status=400)
        except jwt.InvalidTokenError:
            return JsonResponse({'error': 'Invalid token'}, status=400)
```

2. 更新 `urls.py`:

```python
from django.urls import path
from . import views

urlpatterns = [
    path('sso/login/', views.SSOLoginView.as_view(), name='sso_login'),
]
```

3. 配置 `settings.py`:

```python
CORS_ALLOWED_ORIGINS = [
    "https://artdott.com",
]

SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
```

## 使用方法

1. 在 artdott.com 网站中添加跳转按钮：

```html
<button onclick="window.location.href='{% url 'redirect_to_b' %}'">
    跳转到B网站
</button>
```

2. 用户点击按钮后，系统将：
   - 生成包含用户信息的 JWT token
   - 重定向到 curiosityroom.xrexp.io 网站
   - curiosityroom.xrexp.io 网站验证 token 并自动登录用户

## 安全建议

1. 在生产环境中：
   - 使用环境变量存储共享密钥
   - 确保使用 HTTPS
   - 设置合适的 token 过期时间
   - 实现请求频率限制
   - 记录详细的日志

2. 定期：
   - 轮换共享密钥
   - 检查 token 使用情况
   - 更新依赖包版本

## 故障排除

常见问题：

1. Token 验证失败
   - 检查两个站点的时间是否同步
   - 验证共享密钥是否一致
   - 确认 token 未过期

2. 跨域请求失败
   - 检查 CORS 配置
   - 确认域名是否在允许列表中

3. 用户信息同步问题
   - 检查数据库连接
   - 验证用户模型字段匹配



## 更新日志

- v1.0.0 (2025-04-14)
  - 初始版本发布
  - 实现基本的跨站点登录功能
  - 添加安全验证机制
