# Проверка прав доступа в DRF

В Django Rest Framework (DRF) разрешения (permissions) позволяют контролировать доступ к различным частям вашего API на основе ролей пользователей, типов запросов и других факторов. Разрешения позволяют вам определять, какие пользователи имеют доступ к определенным представлениям (views) или объектам в вашем API.

## Встроенные разрешения

DRF предоставляет несколько встроенных разрешений, которые можно использовать для настройки доступа к вашему API:

- **IsAuthenticated (Аутентифицирован)**: 

Это базовое разрешение, которое требует, чтобы пользователь был аутентифицирован, чтобы получить доступ к представлению или объекту.

- **IsAdminUser (Администратор):**

Это разрешение позволяет доступ только администраторам. Пользователи, имеющие статус is_staff=True, считаются администраторами.

- **AllowAny (Любой):** 

Это разрешение позволяет доступ к представлению или объекту без аутентификации. Используйте его, если вы хотите, чтобы ваше представление было доступно для всех.

- **IsAuthenticatedOrReadOnly (Аутентифицирован или только для чтения):**

Это разрешение позволяет только аутентифицированным пользователям выполнять операции записи (например, создание, обновление, удаление), в то время как анонимным пользователям разрешено только чтение.

## Пользовательские разрешения (custom permissions)

**Пользовательские разрешения (Custom Permissions)** в Django Rest Framework (DRF) позволяют вам определять собственную логику доступа к вашему API, учитывая более специфичные требования вашего приложения. Вы можете создать собственные разрешения, которые будут проверять различные аспекты запроса или объекта, такие как атрибуты пользователя, свойства объекта и другие факторы.

Для создания пользовательского разрешения в DRF, вам нужно наследоваться от класса `permissions.BasePermission` и реализовать метод `has_permission()` для проверки доступа к представлениям или метод `has_object_permission()` для проверки доступа к отдельным объектам.

Вот некоторые примеры реализации пользовательских разрешений: 

- ### Доступ к представлению только администраторам:

```python
from rest_framework import permissions

class IsAdminUserOnly(permissions.BasePermission):
    """
    Пользовательское разрешение, которое разрешает доступ только администраторам.
    """

    def has_permission(self, request, view):
        # Проверяем, является ли пользователь администратором
        return request.user and request.user.is_staff
```
Это пользовательское разрешение `IsAdminUserOnly` проверяет, является ли текущий пользователь администратором. Если пользователь является администратором `(is_staff=True)`, метод `has_permission()` возвращает `True`, что разрешает доступ к представлению. В противном случае доступ будет отклонен.

- ### Разрешение на чтение только для аутентифицированных пользователей:

```python
from rest_framework import permissions

class IsAuthenticatedReadOnly(permissions.BasePermission):
    """
    Разрешение на чтение только для аутентифицированных пользователей.
    """

    def has_permission(self, request, view):
        return request.user and request.user.is_authenticated
```

Это разрешение разрешает доступ только аутентифицированным пользователям для чтения данных или создания объекта, но не для изменения или удаления.

- ### Разрешение на основе атрибутов объекта:

Предположим, у вас есть модель `Task`, и вы хотите разрешить только создателю задачи изменять или удалять ее:

```python
from rest_framework import permissions

class IsOwnerOrReadOnly(permissions.BasePermission):
    """
    Разрешение, позволяющее доступ только владельцу объекта для изменения или удаления.
    """

    def has_object_permission(self, request, view, obj):
        # Разрешено только для методов GET, HEAD и OPTIONS
        if request.method in permissions.SAFE_METHODS:
            return True
        
        # Разрешить доступ только создателю объекта
        return obj.owner == request.user
```

В этом примере `IsOwnerOrReadOnly` разрешает доступ только владельцу задачи для изменения или удаления. Для методов чтения `(GET, HEAD, OPTIONS)` доступ разрешен всем.

- ### Разрешение на основе групп пользователя:

```python
from rest_framework import permissions

class IsInGroup(permissions.BasePermission):
    """
    Разрешение, позволяющее доступ только пользователям, находящимся в определенной группе.
    """

    def has_permission(self, request, view):
        required_groups = getattr(view, 'required_groups', [])
        user_groups = request.user.groups.values_list('name', flat=True)
        return any(group in user_groups for group in required_groups)
```

Используйте это разрешение, установив список групп, которым пользователь должен принадлежать для доступа к представлению:

```python
class MyView(APIView):
    permission_classes = [IsInGroup]
    required_groups = ['Admins']
```

- ### Разрешение на доступ только для чтения для неаутентифицированных пользователей.

```python
from rest_framework import permissions

class ReadOnlyIfNotAuthenticated(permissions.BasePermission):
    """
    Пользовательское разрешение, которое разрешает только чтение для неаутентифицированных пользователей.
    """

    def has_permission(self, request, view):
        # Проверяем, является ли пользователь аутентифицированным
        if request.user.is_authenticated:
            return True
        # Разрешаем только GET-запросы для неаутентифицированных пользователей
        return request.method in permissions.SAFE_METHODS
```

Это разрешение разрешает только чтение для неаутентифицированных пользователей. Для аутентифицированных пользователей они будут иметь доступ ко всем методам, но для неаутентифицированных пользователей доступны только GET-запросы.

## Установка разрешения

Чтобы установить разрешение в представлении (вью) в Django Rest Framework, вы можете использовать атрибут `permission_classes`. Этот атрибут позволяет указать список разрешений, которые должны быть выполнены для доступа к представлению. Вот как это работает:

```python
rom rest_framework.views import APIView
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response

class MyView(APIView):
    permission_classes = [IsAuthenticated]  # Установка разрешения на доступ только аутентифицированным пользователям

    def get(self, request, format=None):
        # Ваша логика представления
        return Response({"message": "Вы аутентифицированы и имеете доступ к этому представлению."})
```

В этом примере установлено разрешение IsAuthenticated, которое требует, чтобы пользователь был аутентифицирован для доступа к представлению. Если пользователь не аутентифицирован, он получит ошибку аутентификации.

Вы также можете установить несколько разрешений, и все они должны быть выполнены для доступа к представлению. Например:

```python
from rest_framework.views import APIView
from rest_framework.permissions import IsAuthenticated, IsAdminUser
from rest_framework.response import Response

class MyView(APIView):
    permission_classes = [IsAuthenticated, IsAdminUser]  # Установка разрешений на доступ только аутентифицированным администраторам

    def get(self, request, format=None):
        # Ваша логика представления
        return Response({"message": "Вы аутентифицированы и являетесь администратором."})
```

В этом примере доступ к представлению имеют только пользователи, которые аутентифицированы и являются администраторами.

Также есть возможность устанавливать разрешения в методе `get_permissions()`, переопределяя его логику.

Метод `get_permissions()` используется в Django Rest Framework для получения списка разрешений, которые применяются к определенному представлению (view). Этот метод используется в представлениях, которые требуют динамического определения списка разрешений на основе конкретного запроса или других параметров.

Обычно метод `get_permissions() `переопределяется в классе представления (view) для динамического определения списка разрешений. Этот метод возвращает список экземпляров разрешений, которые будут применены к представлению при обработке запроса.

Вот пример использования метода `get_permissions()` в представлении:

```python
from rest_framework.views import APIView
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response

class MyView(APIView):
    def get_permissions(self):
        """
        Получение списка разрешений для данного представления.
        """
        if self.request.method == 'GET':
            # Если метод запроса GET, применяем разрешение только для чтения
            return [IsAuthenticated()]
        else:
            # Для других методов запроса применяем разрешение для аутентифицированных пользователей
            return [IsAuthenticated()]

    def get(self, request, format=None):
        # Ваша логика представления
        return Response({"message": "Получен GET-запрос."})

    def post(self, request, format=None):
        # Ваша логика представления
        return Response({"message": "Получен POST-запрос."})
```

В этом примере метод `get_permissions()` динамически возвращает список разрешений в зависимости от метода запроса. Если запрос является GET-запросом, используется только разрешение для чтения `IsAuthenticated()`, в то время как для других методов запроса применяется разрешение для аутентифицированных пользователей.

## Разрешения в ViewSet

В `ViewSet` разрешения могут быть определены глобально для всего представления или могут быть определены отдельно для каждого действия представления. Для этого используются атрибуты `permission_classes` и метод `get_permissions()`

### Глобальные разрешения

Глобальные разрешения применяются ко всем методам представления, если явно не указано иное.

```python
from rest_framework import viewsets, permissions

class MyViewSet(viewsets.ModelViewSet):
    queryset = MyModel.objects.all()
    serializer_class = MyModelSerializer
    permission_classes = [permissions.IsAuthenticated]  # Глобальное разрешение для всего представления
    
    # Остальной код представления

```

В этом примере глобальное разрешение IsAuthenticated установлено для всего ViewSet. Это означает, что для всех методов в ViewSet, таких как list, retrieve, create, update, destroy и т.д., будет требоваться аутентификация пользователя.

Глобальные разрешения в ViewSet обычно используются для установки базового уровня безопасности для всего представления. Они могут быть дополнены или переопределены с помощью других разрешений на уровне метода, если требуется более специфичный доступ к определенным действиям.

### Разрешения по действиям

Разрешения по действиям в ViewSet в Django Rest Framework позволяют определять различные разрешения для различных действий, выполняемых в представлении. Это дает гибкость в управлении доступом к различным функциям вашего API в зависимости от типа запроса, роли пользователя и других факторов.

Для определения разрешений по действиям в ViewSet можно использовать метод `get_permissions()`. В этом методе вы можете определить различные наборы разрешений для каждого действия, такие как list, retrieve, create, update, destroy и т.д.

```python
from rest_framework import viewsets, permissions

class MyViewSet(viewsets.ModelViewSet):
    queryset = MyModel.objects.all()
    serializer_class = MyModelSerializer

    def get_permissions(self):
        # Определение разрешений для каждого действия

        # Для действия "список" разрешаем доступ для всех
        if self.action == 'list':
            permission_classes = [permissions.AllowAny]

        # Для действия "получение отдельного объекта" требуем аутентификацию
        elif self.action == 'retrieve':
            permission_classes = [permissions.IsAuthenticated]

        # Для действия "создание объекта" требуем, чтобы пользователь был администратором
        elif self.action == 'create':
            permission_classes = [permissions.IsAdminUser]

        # Для остальных действий используем глобальные разрешения
        else:
            permission_classes = self.permission_classes
        
        return [permission() for permission in permission_classes]
```

В этом примере:

- Для действия `"список" (list)` мы разрешаем доступ для всех, используя `permissions.AllowAny`.
- Для действия `"получение отдельного объекта" (retrieve)` мы требуем аутентификацию с помощью `permissions.IsAuthenticated`.
- Для действия `"создание объекта" (create)` мы требуем, чтобы пользователь был администратором с помощью `permissions.IsAdminUser`.
- Для всех остальных действий используются глобальные разрешения, определенные в атрибуте `permission_classes`.