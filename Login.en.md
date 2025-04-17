# Django Cross-Site Automatic Login Solution

This project implements automatic login functionality between two Django websites. After users log in to artdott.com, they can automatically redirect and log in to curiosityroom.xrexp.io by clicking a button. This solution uses JWT (JSON Web Token) to implement secure cross-site authentication.

## Features

- JWT-based secure authentication
- Automatic user information synchronization
- Simple and easy-to-use interface
- HTTPS support
- Cross-origin request support

## Tech Stack

- Python 3.8+
- Django 3.2+
- PyJWT 2.6.0
- django-cors-headers 3.14.0

## Installation Steps

1. Clone the project locally

```bash
git clone <your-repository-url>
cd <project-directory>
```

2. Create and activate virtual environment

```bash
python -m venv venv
source venv/bin/activate  # Linux/Mac
# or
.\venv\Scripts\activate  # Windows
```

3. Install dependencies

```bash
pip install -r requirements.txt
```

4. Configure environment variables

Create `.env` files in the root directory of both projects:

```env
SECRET_KEY=your-shared-secret-key
DEBUG=False
ALLOWED_HOSTS=your-domain.com
```

## Configuration Guide

### artdott.com (Source Site) Configuration

1. Add view code to `views.py`:

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

2. Update `urls.py`:

```python
from django.urls import path
from . import views

urlpatterns = [
    path('redirect-to-b/', views.redirect_to_target_site, name='redirect_to_b'),
]
```

### curiosityroom.xrexp.io (Target Site) Configuration

1. Add view code to `views.py`:

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

2. Update `urls.py`:

```python
from django.urls import path
from . import views

urlpatterns = [
    path('sso/login/', views.SSOLoginView.as_view(), name='sso_login'),
]
```

3. Configure `settings.py`:

```python
CORS_ALLOWED_ORIGINS = [
    "https://artdott.com",
]

SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
```

## Usage

1. Add redirect button in artdott.com:

```html
<button onclick="window.location.href='{% url 'redirect_to_b' %}'">
    Go to curiosityroom.xrexp.io
</button>
```

2. When users click the button, the system will:
   - Generate JWT token containing user information
   - Redirect to curiosityroom.xrexp.io
   - curiosityroom.xrexp.io verifies the token and automatically logs in the user

## Security Recommendations

1. In production environment:
   - Store shared keys in environment variables
   - Ensure HTTPS usage
   - Set appropriate token expiration time
   - Implement request rate limiting
   - Record detailed logs

2. Regularly:
   - Rotate shared keys
   - Monitor token usage
   - Update dependency package versions

## Troubleshooting

Common issues:

1. Token verification failure
   - Check if time is synchronized between sites
   - Verify shared key consistency
   - Confirm token has not expired

2. Cross-origin request failure
   - Check CORS configuration
   - Confirm domain is in allowed list

3. User information synchronization issues
   - Check database connection
   - Verify user model field matching

## Changelog

- v1.0.0 (2025-04-14)
  - Initial release
  - Implemented basic cross-site login functionality
  - Added security verification mechanism
