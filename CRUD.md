# CRUD в DRF

## Что такое CRUD?

**CRUD** - это сокращение, описывающее основные действия, выполняемые в отношении данных в базе данных или системе управления данными. Каждая буква в аббревиатуре CRUD представляет определенную операцию:

**Сreate (Создание):** Операция, связанная с добавлением новых записей или объектов в базу данных.

**Read (Чтение):** Действие, направленное на получение данных из базы данных. Это включает в себя чтение всей базы данных, выборку конкретных записей или выполнение запросов для получения необходимой информации.

**Update (Обновление):** Действие, направленное на изменение существующих данных в базе данных. Это может включать в себя обновление полей записи или изменение значений объекта.

**Delete (Удаление):** Операция, связанная с удалением данных из базы данных. Это может быть удаление целых записей или объектов, а также удаление определенных полей.

Каждое из вышеуказанных действий обычно выполняется своим HTTP методом:

**GET:** Используется для получения данных. Например, чтение списка объектов или конкретного объекта.
**POST:** Используется для создания новых данных. Например, добавление нового объекта.
**PUT:** Используется для полного обновления существующих данных. Например, обновление полей объекта.
**PATCH:** Используется для частичного обновления существующих данных. Только измененные поля передаются для обновления.

Эти методы могут быть использованы в представлениях DRF, таких как `APIView`, `GenericAPIView`, а также в предопределенных классах API, таких как `ListAPIView, CreateAPIView, UpdateAPIView, DestroyAPIView`, и так далее. В зависимости от сценария использования, вы можете выбрать подходящий HTTP-метод для выполнения той или иной операции через ваше API на основе DRF.

Давайте рассмотрим некоторые варианты реализации действий над объектами. За основу возьмём следующую модель:

```python
#models.py
from django.db import models


class Phone(models.Model):
    name = models.CharField(max_length=100)
    ram = models.SmallPositiveIntegerField()
    color = models.CharField(max_length=20)
    price = models.DecimalField(max_digits=10, decimal_places=2)

    def __str__(self):
        return self.name
```

и объявим самый простой сериализатор

```python #
#serializers.py
from rest_framework import serializers

from .models import Phone


class PhoneSerializer(serializers.ModelSerializer):
    class Meta:
        model = Phone
        fields = "__all__"
```

## FBV (вьюшки на основе функций)

Принцип построения FBV в DRF очень похож на то, что мы реализуем в обычном Django-проекте. Для выполнения определённого 
действия необходимо обработать нужный метод при помощи декоратора `api_view`, например:

### Получение списка записей
```python
#views.py
from rest_framework.decorators import api_view
from rest_framework.response import Response

from .models import Phone
from .serializers import PhoneSerializer

@api_view(["GET"])
def phones_list(request):
    # получаем записи из БД
    phones = Phone.objects.all()
    # сериализуем полученные записи, указываем many=True, т.к. передаётся список записей
    serializer = PhoneSerializer(phones, many=True)
    # возвращаем в ответе сериализованные данные
    return Response(serializer.data)
```

привяжем указанную вьюшку в urls.py
```python
#urls.py
...
from django.urls import path

from app.views import phones_list

urlpatterns = [
    ...,
    path("api/v1/phones/", phones_list)
]
```
теперь для получения записей клиенту необходимо выполнить GET-запрос по адресу `http://localhost:8000/api/v1/phones/`. В ответ
он получит json в виде списка записей:
```json
[
  {
    "id": 1,
    "name": "Apple Iphone 15 Pro Max",
    "ram": 8,
    "color": "черный",
    "price": 100000.00
  },
  {
    "id": 2,
    "name": "Samsung Galaxy S23",
    "ram": 8,
    "color": "серый",
    "price": 85000.00
  }
]
```

### Создание записей

Для реализации создания записей, необходимо обработать данные, отправляемые клиентом в запросе, провалидировать их, создать запись и выдать сообщение клиенту

```python
#views.py

...

@api_view(["POST"])
def create_phone(request):
    #получаем данные запроса
    data = request.data
    #передаём полученные данные в сериализатор
    serializer = PhoneSerializer(data=data)
    #проверяем валидны ли данные, если нет, то он выдаст ошибку
    if serializer.is_valid(raise_exception=True):
        #сохраняем объект
        serializer.save()
        #выдаём ответ
        return Response({"message": "Объект успешно создан"})
```

```python
# urls.py
...
from app.views import create_phone 


urlpatterns = [
  ...,
  path("api/v1/phones/create/", create_phone)
]
```

клиенту для создания объекта необходимо будет выполнить POST-запрос по адресу `http://localhost:8000/api/v1/phones/create/`, передав в теле запроса данные по записи, вида:
```json
{
  "name": "Redmi Note 13 Pro",
  "ram": 12,
  "color": "синий",
  "price": 20000.00
}
```
при успешном запросе пользователь получит в ответ сообщение:
```json
{
  "message": "Объект успешно создан"
  }
```

## CBV (вьюшки на основе классов)

DRF предоставляет широкий выбор вариантов построения CRUD c использованием классов вью:

- APIView
- GenericView (готовые классы для реализации определённого действия)
- ViewSet (специальные классы, которые позволяют реализовать все, ну или по крайней мере несколько, действий в одном классе)

Давайте рассмотрим некоторые из этих вариантов:

## APIView

Класс APIView является базовым классом, на основе которых строятся остальные классы вьюшек.

По существу, реализация представления с помощью APIView не сильно отличается от того, как мы строим вью на функциях. 

Так что же необходимо сделать?

Необходимо объявить класс, наследованный от APIView. Внутри него нужно будет реализовать метод, с названием метода запроса, который Вы хотите обработать, например: `get`, `post`, `patch` и т.д.
В остальном, реализация будет такая же, как и в вышеуказанных примерах.

### Получение одной записи по id
```python
...

class PhoneDetailsView(APIView):
    def get(self, request, phone_id):
        #пытаемся получить запись по id
        try:
            phone = Phone.objects.get(id=phone_id)
            # сериализуем полученный объект
            serializer = PhoneSerializer(phone)
            # выдаём в ответе сериализованные данные
            return Response(serializer.data)
        # если объект с указанной id не найден
        except Phone.DoesNotExist:
            # выдаём 404
            return Response({"message": "Запись не найдена"}, status=status.HTTP_404_NOT_FOUND)
```
в urls для этого пути нужно будет указать обязательный параметр `phone_id`, также для привязки класса к пути необходимо вызвать метод as_view()
```python
#urls.py
...
from app.views import PhoneDetailsView


urlpatterns = [
    ...,
    path("api/v1/phones/<int:phone_id>/", PhoneDetailsView.as_view())
]
```

соответственно, клиенту необходимо будет выполнить GET-запрос по адресу `http://localhost:8000/api/v1/phones/<id_записи>/`
В ответ на что он получит запись в виде json:
```json
{
  "id": 2,
  "name": "Samsung Galaxy S23",
  "ram": 8,
  "color": "серый",
  "price": 85000.00
}
```

### Обновление записи

Для обновления записи также необходимо получить id записи, проверить, что такое объект есть в БД. 
Если объект есть, то получаем данные запроса, передаём их в сериализатор, валидируем и если они валидны, то сохраняем.
Если же объекта нет, то выбросим 404

```python
#views.py
...


class PhoneUpdateView(APIView):
    def patch(self, request, phone_id):
        #пытаемся получить запись по id
        try:
            phone = Phone.objects.get(id=phone_id)
            #получаем обновлённые данные из запроса
            data = request.data
            # сериализуем полученный объект
            serializer = PhoneSerializer(phone, data=data)
            if serializer.is_valid():
                #проверяем валидны ли данные, если да, то сохраняем
                serializer.save()
                # выдаём в ответе сериализованный объект с обновленными данными 
                return Response(serializer.data)
            # если данные не валидны, то выдаём ошибку
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
        # если объект с указанной id не найден
        except Phone.DoesNotExist:
            # выдаём 404
            return Response({"message": "Запись не найдена"}, status=status.HTTP_404_NOT_FOUND)
```

```python
#urls.py
...
from app.views import PhoneUpdateView


urlpatterns = [
    ...,
    path("api/v1/phones/update/<int:phone_id>/", PhoneUpdateView.as_view())
]
```

для обновления объекта пользователю необходимо отправить PATCH-запрос на url `http://localhost:8000/api/v1/phones/update/<int:phone_id>/`, 
передав в теле запроса обновленные данные.
```json
{
  "price": 80000
}
```
в ответ пользователь получит обновленный объект
```json
{
  "id": 2,
  "name": "Samsung Galaxy S23",
  "ram": 8,
  "color": "серый",
  "price": 80000.00
}
```

### Удаление записи

Для обновления записи также необходимо получить id записи, проверить, что такое объект есть в БД.
Если объект есть, то удалить, если нет, то выбросим 404

```python
#views.py
...


class PhoneDeleteView(APIView):
    def delete(self, request, phone_id):
        #пытаемся получить запись по id
        try:
            phone = Phone.objects.get(id=phone_id)
            # если объект есть, то удаляем его
            phone.delete()
            # выдаём пользователю ответ
            return Response({"message": "Запись успешно удалена"})
        # если объект с указанной id не найден
        except Phone.DoesNotExist:
            # выдаём 404
            return Response({"message": "Запись не найдена"}, status=status.HTTP_404_NOT_FOUND)
```

```python
#urls.py
...
from app.views import PhoneDeleteView


urlpatterns = [
    ...,
    path("api/v1/phones/delete/<int:phone_id>/", PhoneDeleteView.as_view())
]
```

Для удаления объекта пользователю необходимо отправить DELETE-запрос на url `http://localhost:8000/api/v1/phones/delete/<int:phone_id>/`,
В случае успешного запроса пользователь получит ответ
```json
{
  "message": "Запись успешно удалена"
}
```

## Generic views

Как Вы уже успели понять, при использовании APIView нам нужно описывать каждое действие самостоятельно. В противовес этому,
есть возможность использовать уже готовые классы для каждого действия (получение списка, получение деталей, создание, обновление, удаление).
Такие классы называются generic view. 

Давайте посмотрим, как они работают:

В основе всех generic классов стоит класс **GenericAPIView**. 

`GenericAPIView`  это базовый класс, который предоставляет общие или универсальные (generic) методы для работы с 
различными типами представлений (views) в DRF. Он упрощает создание представлений, которые выполняют общие задачи, такие 
как получение списка объектов, получение одного объекта, создание, обновление или удаление объекта. 

Также он интегрирует 
множество функциональных возможностей, таких как авторизация, сериализация и дополнительные характеристики, чтобы 
упростить процесс создания представлений.

Для каждого из действий используется свой Generic класс:

- **ListAPIView** - для получения списка
- **RetrieveAPIView** - для получения деталей
- **CreateAPIView** - для создания объекта
- **UpdateAPIView** - для обновления объекта
- **DestroyAPIView** - для удаления объекта

Каждый из вышеуказанных классов наследуется от GenericAPIView, а также каждый наследует свой класс-миксин 
(ListModelMixin, RetrieveModelMixin, CreateModelMixin и т.д.), который предоставляет дополнительные методы для определённого действия.

Generic классы очень просты в использовании. Всё, что требуется сделать - это наследоваться от нужного класса и указать 
`queryset` и `класс сериализации`. Давайте рассмотрим некоторые примеры:

```python
...
from rest_framework import generics


class PhonesListView(generics.ListAPIView):
    queryset = Phone.objects.all()
    serializer_class = PhoneSerializer
    
    
class PhoneDetailsView(generics.RetrieveAPIView):
    queryset = Phone.objects.all()
    serializer_class = PhoneSerializer
    
    
class CreatePhoneView(generics.CreateAPIView):
    queryset = Phone.objects.all()
    serializer_class = PhoneSerializer
    
    
class UpdatePhoneView(generics.UpdateAPIView):
    queryset = Phone.objects.all()
    serializer_class = PhoneSerializer
    
    
class DeletePhoneView(generics.DestroyAPIView):
    queryset = Phone.objects.all()
    serializer_class = PhoneSerializer
```

Это самые базовые варианты использования, и как Вы видите, всё довольно просто. 
Но есть вариант построения CRUD ещё проще, в частности, есть возможность объединить несколько действий в одном классе.

Например, среди Generic классов есть 2 класса: 

- **ListCreateAPIView** - это класс-представление, предоставляющий реализацию обработки HTTP-запросов `GET` (список) и `POST`
(создание) для ресурса. Этот вид представления полезен, когда необходимо предоставить представление, поддерживающее как 
отображение коллекции объектов, так и создание новых объектов в этой коллекции.

- **RetrieveUpdateDestroyAPIView** - это ещё один класс-представление, предоставляющий реализацию обработки HTTP-запросов 
для выполнения операций извлечения `GET`, обновления `PUT/PATCH` и удаления `DELETE` над одним объектом ресурса. Этот 
класс-представление обычно используется для работы с отдельными объектами, а не с коллекциями.

Используются они так же просто, как и классы, используемые выше
```python
#views.py
...
from rest_framework.generics import ListCreateAPIView, RetrieveUpdateDestroyAPIView


class PhoneListCreateView(ListCreateAPIView):
    queryset = Phone.objects.all()
    serializer_class = PhoneSerializer


class PhoneDetailsUpdateDeleteView(RetrieveUpdateDestroyAPIView):
    queryset = Phone.objects.all()
    serializer_class = PhoneSerializer
```

```python
#urls.py
...
from app.views import PhoneListCreateView, PhoneDetailsUpdateDeleteView


urlpatterns = [
    ...,
    path("api/v1/phones/", PhoneListCreateView.as_view()),
    path("api/v1/phones/<int:phone_id>/", PhoneDetailsUpdateDeleteView.as_view())
]
```

Но и это ещё не всё. Есть вариант реализовать все действия CRUD в одном классе и называется он viewset.

## Viewset

ViewSet в Django REST framework - это специальный класс, который помогает вам создавать API для работы с данными в вашем
веб-приложении. Вместо того, чтобы каждый раз писать логику для обработки запросов (например, для создания, обновления 
или удаления данных), вы можете использовать ViewSet, который уже предоставляет эту логику.

В частности, ViewSet может обрабатывать операции создания, чтения, обновления и удаления (CRUD) для ваших данных. 
Вы определяете, как выглядит ваш набор данных, как его представлять (сериализовать) и как обрабатывать запросы, а ViewSet 
заботится о большей части "рутины" кода, связанного с API.

Чаще всего используют класс `ModelViewSet`. ModelViewSet представляет собой предопределенный 
класс-представление, который предоставляет полный набор операций CRUD (Create, Retrieve, Update, Delete) для модели Django. 
Этот класс-представление объединяет функциональность ListModelMixin, CreateModelMixin, RetrieveModelMixin, 
UpdateModelMixin и DestroyModelMixin, предоставляя автоматически сгенерированные представления для работы с моделью данных.

Вот пример использования `ModelViewSet`:

```python
#views.py
from rest_framework.viewsets import ModelViewSet


class PhoneViewSet(ModelViewSet):
    queryset = Phone.objects.all()
    serializer_class = PhoneSerializer
```

Однако, есть небольшая разница между использованием обычных классов и viewset. Пути для viewset обычно формируются при 
помощи специального объекта router.

```python
#urls.py
from django.urls import include, path
from rest_framework.routers import DefaultRouter

from app.views import PhoneViewSet


router = DefaultRouter()
router.register('phones', PhoneViewSet, basename='phones')

urlpatterns = [
    ...,
    path('api/v1/phones/', include(router.urls)),
]
```

 `DefaultRouter` представляет собой встроенный класс, который облегчает создание URL-путей для представлений ViewSet. 
 DefaultRouter автоматически создает URL-пути для стандартных операций CRUD (Create, Retrieve, Update, Delete) на основе 
 предоставленного ViewSet.
 
DefaultRouter автоматически создаст следующие URL-пути для вашего viewset:

- `/api/v1/phones/` - список объектов (GET) и создание нового объекта (POST).
- `/api/v1/phones/id_записи/` - получение (GET), обновление (PUT), частичное обновление (PATCH) и удаление (DELETE) 
- конкретного объекта.

router также поддерживает опцию создания URL-пути для пользовательских методов (action), если они определены в вашем ViewSet

## Заключение

Итак, мы рассмотрели основные варианты реализации CRUD в DRF. Как Вы видите, вариантов много, Вы должны выбирать то, что
больше подходит для конкретной ситуации. Но если подвести краткий итог, то:

- если нужен полный контроль или нужно обрабатывать какие-то нестандартные варианты, то выбирайте FBV или класс APIView
- в случае, если нужно реализовать какие-то конкретные действия, то Ваш выбор - Generic классы
- если же нужно реализовать все действия сразу, то можно использовать viewset.