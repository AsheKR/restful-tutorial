# Serializers

Serializers는 쿼리셋들 및 모델 인스턴스와 같은 복잡한 데이터를 `JSON`, `XML` 또는 기타 컨텐트 유형으로 쉽게 렌더링 할 수 있는 Python 기본 데이터 유형으로 변환해준다. 또한 serializer는 desrialization을 제공하여, 들어오는 데이터의 유효성을 처음 확인한 후에 구문 분석 된 데이터를 복합 형식으로 다시 변환 할 수 있다.

> Django에서 Client로 복잡한 데이터(모델 인스턴스) 를 보내려면 `string`으로 변환해야한다.
> 이 변환을 `Serializer` 라고 한다.
> 반대로 Client의 `string`을 Dajgno로 받을 때 Python 기본 데이터 유형으로 받아야 하는데 이 변환을 `deserializer` 라고 한다

REST 프레임워크의 serializers는 Django의 `Form` 및 `ModelForm` 클래스와 매우 유사하게 작동한다. REST 프레임워크는 `ModelSerializer`(모델 인스턴스와 쿼리셋을 다루는 시리얼라이저를 생성하기 유용한 클래스)뿐만 아니라 응답의 출력을 제어하는 강력하고 일반적인 방법을 제공하는 `Serializer` 클래스를 제공한다.

## DECLARING SERALIZERS

예제를 위해 사용할 간단한 객체를 생성한다.

```python
from datetime import datetime

class Comment(object):
  def __init__(self, email, content, created=None):
    self.email = email
    self.content = content
    self.created = created or datetime.now()
```

`Comment` 객체에 해당하는 serializer 및 deserializer화하는데 사용할 수 있는 `serializer`를 선언한다.

serializer를 선언하면 `form`을 선언하는 것과 매우 유사하다.

```python
from rest_framework import serializers

class CommentSerializer(serializers.Serializer):
  email = serializers.EmailField()
  content = serializers.CharField(max_length=200)
  created = serializer.DateTimeField()
```

## SERIALIZING OBJECTS

`CommentSerializer`를 사용하여 `Comment` 또는 여러 `Comment`를 serializer 할 수 있다.
다시말하면 `Serializer`클래스를 사용하는 것은 `Form`을 사용하는 것과 비슷하다.

```python
comment = Comment(email='ex@ex.com', content='foo bar')
serializer = CommentSerializer(comment)
serializer.data
# {'email': 'ex@ex.com', 'content': 'foo bar', 'created': '<객체 생성시간>'}
```

이 시점에서 모델 인스턴스를 파이썬 기본 유형으로 변환했다. serializer 과정을 마무리하기 위해 데이터를 json으로 렌더링한다.

```python
from rest_framework.renderers import JSONRenderer

json = JSONRenderer().render(serializer.data)
json
# b'{"email": "ex@ex.com", "content": "foo bar", "created": "<객체 생성시간>"}'
```

## DESERIALIZING OBJECTS

`Deserialization`도 비슷하다. 먼저 스트림 데이터를 파이썬 데이터 형식으로 파싱한다.

```python
from django.utils.six import BytesIO
from rest_framework.parsers import JSONParser

stream = BytesIO(json)
data = JSONParser().parse(stream)
```

이후 기본 데이터 유형을 `validated dict` 형식으로 복원한다.

```python
serializer = CommentSerializer(data=data)
serializer.is_valid()
#True
serializer.validated_data
# {'email': 'ex@ex.com', 'content': 'foo bar', 'created': '<객체 생성시간>'}
```

## SAVING INSTANCES

유효성이 검사된 데이터를 기반으로 완전한 객체 인스턴스를 반환하려면 `.create()`, `.update()` 메서드 중 하나나 둘 모두를 구현해야한다.

```python
class CommentSerializer(serializers.Serializer):
    email = serializers.EmailField()
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField()

    def create(self, validated_data):
        return Comment(**validated_data)

    def update(self, instance, validated_data):
        instance.email = validated_data.get('email', instance.email)
        instance.content = validated_data.get('content', instance.content)
        instance.created = validated_data.get('created', instance.created)
        return instance
```

`.save()`를 호출하면 serializer 클래스를 인스턴스화 할 때 기존 인스턴스가 전달되었는지 여부에 따라 새 인스턴스를 만들거나 기존 인스턴스를 업데이트한다.

```python
# .save() will create a new instance.
serializer = CommentSerializer(data=data)

# .save() will update the existing `comment` instance.
serializer = CommentSerializer(comment, data=data)
```

`.create()` 및 `.update()`메서드는 모두 선택사항이다. serializer클래스의 유사 케이스에 따라 하나 또는 둘 모두 구현할 수 있다.

### `.save()`에 추가 속성 전달

때로는 인스턴스를 저장하는 시점에 view에서 데이터를 추가할 수 있어야한다. 이 추가데이터는 현재 사용자 또는 현재시간, 요청 데이터의 일부가 아닌 다른 정보가 포함될 수 있다.

`.save()`를 호출할 때 추가 키워드 인수를 포함시켜 그렇게 할 수 있다.

`serializer.save(owner=request.user)`

추가 키워드 인수는 `.create()` 또는 `.update()`가 호출될 때 `validated_data`인수에 포함된다.

#### `.save()` 재정의

어떤 경우에는 `.create()` 및 `.update()` 메서드가 의미가 없을 수 있다.
예를들어 아래의 `ContactForm`은 새 인스턴스를 생성하지 않고, 대신 이메일이나 메세지를 전달한다.

```python
class ContactForm(serializers.Serializer):
    email = serializers.EmailField()
    message = serializers.CharField()

    def save(self):
        email = self.validated_data['email']
        message = self.validated_data['message']
        send_email(from=email, message=message)
```

이 경우 serializer.validated_data 속성에 직접 액세스 해야한다.

## Validation

데이터를 `deserializer`할 때 유효성이 검사 된 데이터에 액세스 하기 전 항상 `is_valid()`를 호출하거나 객체 인스턴스를 저장해야한다. 유효성 검사 오류가 발생하면, `.errors()` 속성에 결과 오류 메세지를 나타내는 `dict`가 포함된다.

```python
serializer = CommentSerializer(data={'email': 'foobar', 'content': 'baz'})
serializer.is_valid()
# False
serializer.errors
# {'email': [u'Enter a valid e-mail address.'], 'created': [u'This field is required.']}
```

dict의 각 키는 필드이름이며, 값은 해당 필드에 해당하는 오류 메세지의 문자열 목록이다.
`non_field_errors`는 일반적인 유효성 검사 오류를 나열한다. `non_field_errors`키의 이름은 REST 프레임워크 설정의 `NON_FIELD_ERRORS_KEY`를 사용하여 사용자 지정할 수 있다.

리스트를 deserializer할 때 오류는 각 deserializer화 항목을 나타내는 dict 목록으로 반환된다.

### 유효하지 않은 데이터에 대한 예외 발생

`.is_valid()`메서드는 유효성 검사 오류가 있는 경우 `serializer.ValidationError`예외를 발생시키는 선택적 `raise_exception`플래그를 사용한다.

이러한 예외는 RESTT 프레임워크에서 제공하는 기본 예외 처리기에서 자동으로 처리되며, 기본적으로 `HTTP 400 Bad Request`응답을 반환한다.

```python
# Return a 400 response if the data was invalid.
serializer.is_valid(raise_exception=True)
```

### 필드레벨 검증

Serializer 서브 클래스에 `.validate_<field_name>` 메서드를 추가하여 custom 필드 레벨 유효성 검증을 지정할 수 있다. 이것들은 Django form의 `clean_<field_name>`과 비슷하다.

이 메서드는 인수가 필요하며, 유효성 검사가 필요한 필드 값이다.
`validate_<non_field_errors>` 메서드는 유효한 값을 반환하거나 `serializers.ValidationError`를 발생시켜야한다.

```python
from rest_framework import serializers

class BlogPostSerializer(serializers.Serializer):
    title = serializers.CharField(max_length=100)
    content = serializers.CharField()

    def validate_title(self, value):
        """
        Check that the blog post is about Django.
        """
        if 'django' not in value.lower():
            raise serializers.ValidationError("Blog post is not about Django")
        return value
```

`<field_name>`이 `required=False`를 사용하여 `serializer`에 선언된 경우, 필드가 포함되어 있지 않으면 이 유효성 검사 단계가 수행되지 않는다.

### 객체 수준 유효성 검사

여러 필드에 대한 액세스가 필요한 다른 유효성 검사를 하려면, `.validate()` 메서드를 추가한다. 이 메서드는 필드 값이 포함된 dict형태의 단일 인수를 취한다. 필드레벨검증과 같이 `ValidationError`를 발생시키거나 유효성 감사된 값을 반환해야한다.

```python
from rest_framework import serializers

class EventSerializer(serializers.Serializer):
    description = serializers.CharField(max_length=100)
    start = serializers.DateTimeField()
    finish = serializers.DateTimeField()

    def validate(self, data):
        """
        Check that the start is before the stop.
        """
        if data['start'] > data['finish']:
            raise serializers.ValidationError("finish must occur after start")
        return data
```

### VALIDATOR

개별 필드에 `validators=[<validator1>, <validator2> ...]`  의 속성을 추가하여 유효성 검사기에 포함할 수 있다.

```python
def multiple_of_ten(value):
    if value % 10 != 0:
        raise serializers.ValidationError('Not a multiple of ten')

class GameRecord(serializers.Serializer):
    score = IntegerField(validators=[multiple_of_ten])
    ...
```

`Meta` 클래스에 Serializer 전체 필드 데이터 집합에 적용되는 재사용 가능한 유효성 검사기를 포함할 수 있다.

```python
class EventSerializer(serializers.Serializer):
    name = serializers.CharField()
    room_number = serializers.IntegerField(choices=[101, 102, 103, 201])
    date = serializers.DateField()

    class Meta:
        # Each room only has one event per day.
        validators = UniqueTogetherValidator(
            queryset=Event.objects.all(),
            fields=['room_number', 'date']
        )
```

더 많은 정보는 [validators documentation](https://www.django-rest-framework.org/api-guide/validators/)을 참고

## ACCESSING THE INITAL DATA AND INSTANCE

serializer 인스턴스에 초기 객체 또는 쿼리셋을 전달 할 때 객체는 `.instance`로 사용가능하다. 초기 객체가 전달되지 않으면 `.instance` 속성은 `None`이 된다.

데이터를 serializer 인스턴스에 전달할 때 수정되지 않은 데이터는 `.initial_data`로 사용 가능하다. data 키워드 인수가 전달되지 않으면 `.initial_data` 속성이 존재하지 않는다.

### PARTIAL UPDATES

기본적으로 serializer는 모든 필드에 값을 전달해야하며 그렇지 않으면 유효성 검사 오류가 발생한다. 부분적 업데이트를 허용하기 위해 `partial`인수를 사용할 수 있다.

```python
# Update `comment` with partial data
serializer = CommentSerializer(comment, data={'content': u'foo bar'}, partial=True)
```

### DEALING WITH NESTED OBJECTS

위의 예제들은 단순 데이터 유형만을 가진 객체를 다루는 경우 문제가 없지만 객체의 일부 속성이 단순 데이터형이 아닌 복잡한 객체를 표현할 수 있어야하는 경우가 있다.

Serializer 클래스 자체는 Field 유형이며, 한 객체 유형이 다른 객체 유형 내 중첩되어 있는 관계를 나타내는데 사용할 수 있다.

```python
class UserSerializer(serializers.Serializer):
    email = serializers.EmailField()
    username = serializers.CharField(max_length=100)

class CommentSerializer(serializers.Serializer):
    user = UserSerializer()
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField()
```

중첩된 표현이 `None`값을 선택적으로 받아들일 수 있으면 `required=False` 플래그를 속성에 적어주어야 한다.

```python
lass CommentSerializer(serializers.Serializer):
    user = UserSerializer(required=False)  # May be an anonymous user.
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField()
```

마찬가지로 중첩 표현이 list 인 경우 `many=True` 플래그를 전달해야한다.

```python
class CommentSerializer(serializers.Serializer):
    user = UserSerializer(required=False)
    edits = EditItemSerializer(many=True)  # A nested list of 'edit' items.
    content = serializers.CharField(max_length=200)
    created = serializers.DateTimeField()
```

### WRITEABLE NESTED REPRESENTATIONS

데이터의 DESERIALIZER를 지원하는 중첩된 표현을 처리할 때, 중첩된 객체의 오류는 중첩된 객체의 필드 이름 아래 중첩된다.

```python
serializer = CommentSerializer(data=
  {
    'user':
      {
        'email': 'foobar',
        'username': 'doe'
      },
    'content': 'baz'
  }
)
serializer.is_valid()
# False
serializer.errors
# {
#   'user':
#     {
#       'email': [u'Enter a valid e-mail address.']
#     },
#   'created': [u'This field is required.']
# }
```

비슷하게 `.validated_data`속성은 중첩된 데이터 구조를 포함한다.

#### 중첩된 표현을 위한 `.create()`메서드 작성하기

쓰기 가능한 중첩표현을 지원하려면 `.create()`, `.update()`메서드를 작성해야한다.

```python
class UserSerializer(serializers.ModelSerializer):
    profile = ProfileSerializer()

    class Meta:
        model = User
        fields = ('username', 'email', 'profile')

    def create(self, validated_data):
        profile_data = validated_data.pop('profile')
        user = User.objects.create(**validated_data)
        Profile.objects.create(user=user, **profile_data)
        return user
```

#### 중첩된 표현을 위한 `.update()`메서드 작성하기

업데이트의 경우, 관계 업데이트를 처리하는 방법에 대해 신중하게 생각해야한다.
예를들어, 관계에 대한 데이터가 제공되지 않은 경우 어떤 일이 일어나야 하나?

- 관계를 데이터에비스에서 `NULL`로 설정한다.
- 연관 인스턴스를 삭제한다.
- 데이터를 무시하고 인스턴스를 있는 그대로 둔다.
- 유효성 검증 오류를 발생시킨다.

```python
def update(self, instance, validated_data):
       profile_data = validated_data.pop('profile')
       # Unless the application properly enforces that this field is
       # always set, the follow could raise a `DoesNotExist`, which
       # would need to be handled.
       profile = instance.profile

       instance.username = validated_data.get('username', instance.username)
       instance.email = validated_data.get('email', instance.email)
       instance.save()

       profile.is_premium_member = profile_data.get(
           'is_premium_member',
           profile.is_premium_member
       )
       profile.has_support_contract = profile_data.get(
           'has_support_contract',
           profile.has_support_contract
        )
       profile.save()

       return instance
```

중첩된 생성 및 업데이트 동작이 애매할 수 있고, 관련 모델간 복잡한 종속성이 필요할 수 있기 때문에 REST 프레임워크3에서는 이러한 메서드를 항상 명시적으로 작성해야한다. 기본적으로 `ModelSerializer`, `.create()`, `.update()` 메서드는 중첩표현에 대한 지원을 포함하지 않는다.

#### 모델 관리자 클래스에서 관련 인스턴스 저장 처리

serializer에 여러 관련 인스턴스를 저장하는 대신 올바른 인스턴스를 생성하는 custom 모델 관리자 클래스를 작성할 수 있다.

예를들어 User인스턴스와 Profile 인스턴스가 항상 쌍으로 함께 생성되도록하고싶다고 가정할 때, 다음과 같이 custom 매니저 클래스를 작성할 수 있다.

```python
class UserManager(models.Manager):
    ...

    def create(self, username, email, is_premium_member=False, has_support_contract=False):
        user = User(username=username, email=email)
        user.save()
        profile = Profile(
            user=user,
            is_premium_member=is_premium_member,
            has_support_contract=has_support_contract
        )
        profile.save()
        return user
```

이 관리자 클래스는 전보다 훌륭하게 사용자 인스턴스와 프로필 인스턴스가 항상 동시에 생성된다는 사실을 캡슐화한다. serializer 클래스의 `.create()`메서드를 새 관리자 메서드를 사용하도록 다시 작성할 수 있다.

```python
def create(self, validated_data):
    return User.objects.create(
        username=validated_data['username'],
        email=validated_data['email']
        is_premium_member=validated_data['profile']['is_premium_member']
        has_support_contract=validated_data['profile']['has_support_contract']
    )
```

## DEALING WITH MULTIPLE OBJECTS

Serializer 클래스는 여러 serializer 또는 deserializer 처리를 할 수 있다.

### 여러 객체 Serializer

단일 객체 인스턴스 대신 쿼리셋 또는 객체 목록을 serializer 하려면 serializer를 인스턴스화 할 때  `many=True` 플래그를 전달해야한다. 그런 다음 serializer할 쿼리셋이나 객체 목록을 전달한다.

```python
queryset = Book.objects.all()
serializer = BookSerializer(queryset, many=True)
serializer.data
# [
#     {'id': 0, 'title': 'The electric kool-aid acid test', 'author': 'Tom Wolfe'},
#     {'id': 1, 'title': 'If this is a man', 'author': 'Primo Levi'},
#     {'id': 2, 'title': 'The wind-up bird chronicle', 'author': 'Haruki Murakami'}
# ]
```

### 여러 객체 Deserializer

여러 객체를 deserializer화하는 기본동작은 생성은 지원하지만 업데이트는 지원하지 않는다.
더 자세한내용은 [ListSerializer](https://www.django-rest-framework.org/api-guide/serializers/#listserializer)를 참조한다.

## INCLUDING EXTRA CONTEXT

serializer되고 있는 객체에 추가로, serializer에 여분의 context를 제공할 필요가 있는 경우가 있다. 한가지 일반적인 경우는 하이퍼링크 된 관계를 포함하는 serializer를 사용하는 경우이며, serializers가 현재 요청에 액세스하여 정규화된 URL을 제대로 생성할 수 있어야한다.

serializer를 인스턴스화 할 때 `context` 인수를 전달하여 임의의 추가 컨텍스트를 제공할 수 있다.

```python
serializer = AccountSerializer(account, context={'request': request})
serializer.data
# {'id': 6, 'owner': u'denvercoder9', 'created': datetime.datetime(2013, 2, 12, 09, 44, 56, 678870), 'details': 'http://example.com/accounts/6/details'}
```

context의 dict는 사용자정의 `.to_representation()`메서드와 같은 serializer 필드 로직 내 `self.context` 속성에 액세스 하여 사용할 수 있다.

# ModelSerializer

Django 모델 정의와 밀접하게 매핑되는 Serializer 클래스
`ModelSerializer`클래스는 모델 필드에 해당하는 필드가 있는 Serializer 클래스를 자동으로 만들 수 있는 ShortCut을 제공한다.

`ModelSerializer`클래스는 다음을 제외하고는 일반적 Serializer클래스와 동일하다.
- 모델을 기반으로 일련의 필드가 자동생성된다.
- unique_together validator와 같은 serializer에 대한 validator를 자동으로 생성한다.
- `.create()`와 `.update()`의 간단한 기본 구현을 포함한다.

```python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = ('id', 'account_name', 'users', 'created')
```

기본적으로 클래스의 모든 모델 필드는 해당 serializer 필드에 매핑된다.

모델의 외래키같은 관계는 `PrimaryKeyRelatedField`에 매핑된다.
[Serializer Relation](https://www.django-rest-framework.org/api-guide/relations/)에 명시된대로 명시적으로 포함되지 않으면 기본적으로 역관계가 포함되지 않는다.

## INSPACTING A `MODELSERIALIZER`

Serializer 클래스는 유용한 필드 표현 문자열을 생성하므로, 필드의 상태를 완전히 검사할 수 있다. 이는 자동으로 생성되는 필드 및 유효성 검사기들을 결정하려는 `ModelSerializer`로 작업할 때 특히 유용하다.

```python
>>> from myapp.serializers import AccountSerializer
>>> serializer = AccountSerializer()
>>> print(repr(serializer))
# AccountSerializer():
#     id = IntegerField(label='ID', read_only=True)
#     name = CharField(allow_blank=True, max_length=100, required=False)
#     owner = PrimaryKeyRelatedField(queryset=User.objects.all())
```

## SPECIFYING WHICH FIELDS TO INCLUDE

기본 모델의 하위 집합을 `ModelSerializer`에서만 사용하려는 경우 `ModelForm`과 마찬가지로 필드를 사용하거나 제거할 수 없다. `fields` 속성을 사용하여 serializer해야하는 모든 필드를 명시적으로 설정하는 것이 좋다. 이렇게하면 모델이 변경될 때 실수로 데이터가 노출될 가능성이 줄어든다.

```python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = ('id', 'account_name', 'users', 'created')
```

`__all__`을 사용하여 모델의 모든 필드를 사용함을 나타낼 수 있다.

```python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = '__all__'
```

혹은 `exclude`를 통해 모든 필드중에서 선택제외할 수 있다.

```python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        exclude = ('users',)
```

## SPECIFYING NESTED SERIALIZATION

기본 `ModelSerializer`는 관계에 기본키를 사용하지만 `depth`옵션을 사용하여 중첩된 표현을 쉽게 생성할 수 있다.

```python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = ('id', 'account_name', 'users', 'created')
        depth = 1
```

## SPECIFYING FIELDS EXPLICITLY
`Serializer` 클래스에서와 마찬가지로 `ModelSerializer`에 확장 필드를 추가하거나 필드를 선언하여 기본 필드를 재정의할 수 있다.

```python
class AccountSerializer(serializers.ModelSerializer):
    url = serializers.CharField(source='get_absolute_url', read_only=True)
    groups = serializers.PrimaryKeyRelatedField(many=True)

    class Meta:
        model = Account
```

확장 필드는 모델의 프로퍼티 또는 호출가능한 메서드일 수 있다.

## SPECIFYING READ ONLY FIELDS

여러 필드를 읽기전용으로 지정할 수 있다. 각 필드를 read_only=True 특성으로 명시적으로 추가하는 대신 `ShortCut`으로 `Meta`클래스의 옵션인 `read_only_fields`를 사용할 수 있다.

```python
class AccountSerializer(serializers.ModelSerializer):
    class Meta:
        model = Account
        fields = ('id', 'account_name', 'users', 'created')
        read_only_fields = ('account_name',)
```

모델의 `editable=False` 및 `AutoField`의 필드인 경우 기본적으로 읽기 전용이므로 따로 추가해줄 필요 없다.

---
읽기 전용 필드가 모델 수준에서 `unique_together` 제약조건의 일부인 특별한 경우가 있다.
이 경우 필드는 제약조건의 유효성을 검사하기 위해 serializer 클래스에서 필요하지만 사용자가 직접 편집할 수는 없어야한다.

이를 처리하는 올바른 방법은 `read_only=True` 및 `default=...` 키워드 인수를 제공하여 serializer에서 필드를 명시적으로 지정하는 것이다.

한가지 예는 다른 식별자로 고유한 현재 인증된 사용자에 대한 읽기전용 관계이다. 이 경우 사용자 필드를 다음과 같이 선언한다.

```python
user = serializers.PrimaryKeyRelatedField(read_only=True, default=serializers.CurrentUserDefault())
```
---

## Additional Keyword arguments

또한 `extra_kwargs` 옵션을 사용하여 필드에 임의의 추가 키워드 인수를 지정할 수 있다. read_only_fields와 마찬가지로, 이것은 serializer에서 필드를 명시적을 선언할 필요가 없다는 것을 의미한다.

```python
class CreateUserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ('email', 'username', 'password')
        extra_kwargs = {'password': {'write_only': True}}

    def create(self, validated_data):
        user = User(
            email=validated_data['email'],
            username=validated_data['username']
        )
        user.set_password(validated_data['password'])
        user.save()
        return user
```

## Relational Fields

모델 인스턴스를 serializer 할 때 관계를 나타내기 위해 선택할 수 있는 여러 방법들이 있다. `ModelSerializer`의 기본 표현은 관련 인스턴스의 기본 키를 사용하는 것이다.
다른 표현은 하이퍼링크를 사용하여 serializer, 완전한 중첩 표현을 serializer, 사용자 정의 표현을 사용하여 serilizer하는 것을 포함한다.

## CUSTOMIZING FILED MAPPINGS

ModelSerializer 클래스는 serializer를 인스턴스화 할 때 serializer 필드가 자동으로 결정되는 방식을 변경하기 위해 재정의할 수 있는 API도 제공한다.
일반적으로 ModelSerializer가 기본적으로 필요한 필드를 생성하지 않으면 클래스에 명시적으로 추가하거나 대신 일반 Serializer클래스를 사용해야한다. 그러나 경우에 따라 특정 모델에 대해 serializer 필드가 생성되는 방식을 정의하는 새 기본 클래스를 만들 수도 있다.

`.serializer_field_mapping`
Django 모델 클래스와 REST 프레임워크 serializer 클래스의 매핑
이 맵핑을 겹쳐쓰면 각 모델 클래스에 사용해야 하는 기본 serializer 클래스를 변경할 수 있다.

`.serializer_related_field`
이 속성은 기본적으로 관계형 필드에 사용되는 serializer 필드 클래스여야 한다.
`ModelSerializer`의 경우 기본값은 `PrimaryKeyRelatedField`이다.
`HyperlinkedModelSerializer`의 경우 기본값은 `serilaizer.HyperRelatedField`이다.

`serializer_url_field`
serializer의 `url`필드에 사용해야하는 serializer 필드 클래스이다. `serializer.HyperlinkedIdentityField`가 기본값이다.

`serializer_choice_field`
serializer의 choice 필드에 사용해야하는 serializer 필드 클래스이다.
serializer.ChoiceFields가 기본 값이다.

### THE FIELD_CLASS AND FIELD_KWARGS API

다음 메서드는 serializer에 자동으로 포함되어야하는 각 필드의 클래스 및 키워드 인수를 결정하기 위해 호출된다. 이 메서드들은 각각 `(field_class, field_kwargs)`의 튜플을 리턴한다.

`.build_standard_field(self, field_name, model_field)`
표준 모델 필드에 매핑되는 serializer 필드를 생성하기 위해 호출된다. 디폴트의 구현은 `serializer_field_mapping`속성에 근거한 serializer 클래스를 돌려준다.

`.build_relational_field(self, field_name, relation_info)`
관계형 모델 필드에 매핑되는 serializer 필드를 생성하기 위해 호출된다.
디폴트의 구현은 `serializer_relational_field`속성에 근거한 serializer 클래스를 돌려준다.
`relation_info` 인수는 명명된 튜플이며 `model_field`, `relation_model`, `to_many` 및 `has_through_model`속성을 포함한다.

`.build_nested_field(self, field_name, relation_info, nested_depth)`
`depth` 옵션이 설정된 경우, 관계형 모델 필드에 매핑되는 serializer 필드를 생성하기 위해 호출된다.
기본 구현은 `ModelSerializer` 또는 `HyperlinkedModelSerializer`를 기반으로 중첩된 Serializer 클래스를 동적으로 만든다.
`nested_depth`는 `depth`옵션의 값에서 1을 뺀 값이다.
`relation_info` 인수는 명명된 튜플이며 `model_field`, `relation_model`, `to_many` 및 `has_through_model`속성을 포함한다.

`.build_url_field(self, field_name, model_class)`
serializer 자신의 `url`필드에 대한 serializer 필드를 생성하기 위해 호출된다. 기본 구현은 `HyperlinkedIdentityField`클래스를 반환한다.

`.build_url_unkown_field(self, field_name, model_class)`
필드 이름이 모델 필드 또는 모델 속성에 매핑되지 않았을 때 호출된다. 서브 클래스에의해 이 동작을 사용자 정의해도, 기본 구현에서는 에러가 발생한다.

# HYPERLINKEDMODELSERIALIZER

`HyperlinkedModelSerializer` 클래스는 기본 키가 아닌 관계 키를 나타내기위해 하이퍼링크를 사용한다는 점을 제외하고는 `ModelSerializer`와 유사하다.
기본적으로 serilizer에는 기본키 필드 대신 `url`필드가 포함된다.
`url` 필드는 `HyperlinkedIdentityField serializer` 필드를 사용하여 표현되며 모델의 모든 관계는 `HyperlinkedRelatedField` serializer 필드를 사용하여 표시된다.

```python
class AccountSerializer(serilaizer.HyperlinkedModelSerializer):
  class Meta:
    model = Account
    fields = ('url', 'id', 'account_name', 'users', 'created')
```

## ABSOLUTED AND RELATIVE URLS

`HyperlinkedModelSerializer`를 인스턴스화 할때는 현재 `request`를 serializer 컨텍스트에 포함해야한다.

`serializer = AccountSerializer(queryset, context={'request': request})`

이렇게하면 하이퍼링크에 적절한 호스트 이름이 포함될 수 있으므로 결과 표현은 다음과 같은 정규화된 URL을 사용한다.

`http://api.example.com/accounts/1/`

상대 경로를 사용하게하기위해서는 serializer 컨텍스트에서 `{'request': None}`을 명시적으로 전달해야한다.

## HOW HYPERLINKED VIEWS ARE DETERMINED

모델 인스턴스에 하이퍼링크하기 위해 어떤 뷰를 사용해야하는지 결정할 수 있는 방법이 필요하다.
기본적으로 하이퍼링크는 `{model_name}-detail`스타일과 이름이 일치해야하며 `pk` 키워드 인수로 특정 인스턴스를 찾는다.
다음과 같이 `extra_kwargs`설정에서 `view_name`과 `lookup_field` 옵션 중 하나 또는 둘 모두를 사용하여 view 이름 및 lookup_field를 사용자 지정할 수 있다.

```python
class AccountSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = Account
        fields = ('account_url', 'account_name', 'users', 'created')
        extra_kwargs = {
            'url': {'view_name': 'accounts', 'lookup_field': 'account_name'},
            'users': {'lookup_field': 'username'}
        }
```

또는 serializer에서 필드를 명시적으로 설정할 수 있다.

```python
class AccountSerializer(serializers.HyperlinkedModelSerializer):
    url = serializers.HyperlinkedIdentityField(
        view_name='accounts',
        lookup_field='slug'
    )
    users = serializers.HyperlinkedRelatedField(
        view_name='user-detail',
        lookup_field='username',
        many=True,
        read_only=True
    )

    class Meta:
        model = Account
        fields = ('url', 'account_name', 'users', 'created')
```

---
하이퍼 링크로 표시된 표현과 URL conf를 적절하게 일치시키는 것은 때로는 약간의 실수일수 있다. `HyperlinkedModelSerializer` 인스턴스의 `repr`을 출력해보는것은 관계가 매핑할 것으로 예상되는 뷰 이름과 조회 필드를 정확하게 검사하는데 특히 유용하다.
---

## Changing the URL field name

URL 입력란의 이름은 `url`로 기본 설정된다. `URL_FIELD_NAME`을 설정을 사용하여 설정을 전체적으로 재정의할 수 있다.

# ListSerializer

`ListSerializer` 클래스는 여러 개체를 한번에 serilize 하고 유효성을 검사하는 동작을 제공한다.
일반적으로 `ListSerializer`를 직접사용할 필요는 없지만 대신 serializer를 인스턴스화할 때 `many=True`를 전달해야한다.
serilizer가 인스턴스화되고 `many=True`가 전달되면 `ListSerializer` 인스턴스가 만들어진다. 그런다음 serializer 클래스는 부모 `ListSerializer`의 자식이된다.
다음 인수는 `Listserializer`필드나 `many=True`로 전달되는 serializer에도 전달할 수 있다.
`allow_empty`
기본적으로 `True`이지만 빈 입력을 허용하려면 `False`로 설정할 수 있다.

## CUSTOMIZING LISTSERIALIZER BEHAVIOR

`ListSerializer`동작을 사용자 정의하려는 경우가 몇가지 있다.
- 특정 요소가 목록의 다른 요소와 충돌하지 않는지 확인하는 등 목록의 특정 유효성 검사를 제공하려고한다.
- 여러 객체의 작성 또는 업데이트 동작을 사용자 정의하려고한다.

이 경우 serializer 메타 클래스에서 `list_serializer_class` 옵션을 사용하여 `many=True`가 전달될 때 사용되는 클래스를 수정할 수 있다.

```python
class CustomListSerializer(serializers.ListSerializer):
    ...

class CustomSerializer(serializers.Serializer):
    ...
    class Meta:
        list_serializer_class = CustomListSerializer
```

## CUSTOMIZING MULTIPLE CREATE

여러 객체 생성을 위한 기본 구현 목록의 각 항목에 대해 `.create()`를 호출한다. 이 동작을 사용자 정의하려면 `many=True`가 전달될 때 사용되는 `list_serializer_class`에서 `.create()`메서드를 사용자정의한다.

```python
class BookListSerializer(serializers.ListSerializer):
    def create(self, validated_data):
        books = [Book(**item) for item in validated_data]
        return Book.objects.bulk_create(books)

class BookSerializer(serializers.Serializer):
    ...
    class Meta:
        list_serializer_class = BookListSerializer
```

## CUSTOMIZING MULTIPLE UPDATE

기본적으로 `ListSerializer` 클래스는 다중 업데이트를 지원하지 않는다. 이는 삽입 및 삭제에 대해 예상되는 동작이 모호하기 때문이다.
여러 업데이트를 지원하려면 명시적으로 업데이트해야한다. 여러 개의 업데이트 코드를 작성할 때 여러 사항을 염두해 두어야한다.

인스턴스 serializer에 명시적으로 `id` 필드를 추가해야한다. 기본적으로 생성된 `id`필드는 `read_only`로 표시됩니다. 이로인해 업데이트시 제거된다. 명시적으로 선언하면 목록 serializer의 `update`메서드에서 사용할 수 있다. 다음은 여러 업데이트를 구현하는 방법에 대한 예이다.

```python
class BookListSerializer(serializers.ListSerializer):
    def update(self, instance, validated_data):
        # Maps for id->instance and id->data item.
        book_mapping = {book.id: book for book in instance}
        data_mapping = {item['id']: item for item in validated_data}

        # Perform creations and updates.
        ret = []
        for book_id, data in data_mapping.items():
            book = book_mapping.get(book_id, None)
            if book is None:
                ret.append(self.child.create(data))
            else:
                ret.append(self.child.update(book, data))

        # Perform deletions.
        for book_id, book in book_mapping.items():
            if book_id not in data_mapping:
                book.delete()

        return ret

class BookSerializer(serializers.Serializer):
    # We need to identify elements in the list using their primary key,
    # so use a writable field here, rather than the default which would be read-only.
    id = serializers.IntegerField()

    ...
    id = serializers.IntegerField(required=False)

    class Meta:
        list_serializer_class = BookListSerializer
```

## CUSTOMIZING LISTSERIALIZER INITIALIZATION

`many=True`가 있는 serializer가 인스턴스화되면 자식 serializer 클래스와 상위 `ListSerializer`클래스 모두에 대해 `.__init__()`메서드에 전달할 인수 및 키워드 인수를 결정해야한다.
디폴트 구현은, `validator`를 제외 한 양쪽 모두의 클래스에 모든 인수를 건네주는것이다. 양쪽 모두 customizer 키워드의 인수이다. 양쪽 모두, 아이디 serializer 클래스를 대상으로하고있다. 때때로 `many=True`가 전달될 때 하위 클래스와 부모 클래스의 인스턴스화 방법으로 명시적으로 지정해야할 때 `many_init`클래스 메서드를 사용하여 해결할 수 있다.

```python
@classmethod
def many_init(cls, *args, **kwargs):
    # Instantiate the child serializer.
    kwargs['child'] = cls()
    # Instantiate the parent list serializer.
    return CustomListSerializer(*args, **kwargs)
```

# BASESERIALIZER

`BaseSerializer`는 serializer와 deserializer 스타일을 쉽게 지원하는데 사용할 수 있는 대안이다. 이 클래스는 Serializer 클래스와 같은 기본 API를 구현한다.

- `.date`
- `.is_valid()`
- `.validated_data`
- `.errors`
- `.save()`

serializer 클래스에서 지원할 기능에 따라 오버라이딩 있는 네가지 메서드가 있다.

- `.to_representation()` - 읽기 조작
- `.to_internal_value()` - 쓰기 조작
- `.create()`, `.update()` - 인스턴스 저장

## READ-ONLY `BASESERIALIZER` CLASSES

이 클래스는 Serializer 클래스와 동일한 인터페이스를 제공하기 때문에 일반 `Serializer` 또는 `ModelSerializer`에서 사용하던 것과 똑같이 기존 CBV와 함께 사용할 수 있다.
`BaseSerializer` 클래스는 browsable API에서 HTML 양식을 생성하지 않는다. 이는 반환하는 데이터에 각 필드를 적절한 HTML 입력으로 렌더링 할 수 있는 모든 필드 정보를 포함하지 않기 때문이다.

```python
class HighScore(models.Model):
    created = models.DateTimeField(auto_now_add=True)
    player_name = models.CharField(max_length=10)
    score = models.IntegerField()
```

`HighScore` 인스턴스를 원시 데이터 유형으로 변환하기 위한 읽기전용 serializer 컨버터를 만든다.

```python
class HighScoreSerializer(serializers.BaseSerializer):
    def to_representation(self, obj):
        return {
            'score': obj.score,
            'player_name': obj.player_name
        }
```

이제 이 클래스를 사용하여 단일 `HighScore` 인스턴스를 serializer 할 수 있다.

```python
@api_view(['GET'])
def high_score(request, pk):
    instance = HighScore.objects.get(pk=pk)
    serializer = HighScoreSerializer(instance)
    return Response(serializer.data)
```

또는 이를 사용하여 여러 인스턴스를 serializer할 수 있다.

```python
@api_view(['GET'])
def all_high_scores(request):
    queryset = HighScore.objects.order_by('-score')
    serializer = HighScoreSerializer(queryset, many=True)
    return Response(serializer.data)
```

## READ-WRITE `BASESERIALIZER` CLASSES

읽기-쓰기 serializer를 만들려면 `.to_internal_value()`메서드를 구현해야한다. 이 메서드는 객체 인스턴스를 구성하는데 사용 될 유효성이 있는 값을 반환하고 제공된 데이터가 잘못된 형식인 경우 `ValidationError`를 발생시킬 수 있다.
`.to_internal_value`를 구현하려면 serializer에서 기본 유효성 검사 API를 사용할 수 있으며 `.is_valid()`, `.validated_data` 및 `.errors`를 사용할 수 있다. `.save()`도 지원하려면 `.create()` 및 `.update()` 메서드 중 하나 또는 모두를 구현해야한다.

```python
class HighScoreSerializer(serializers.BaseSerializer):
    def to_internal_value(self, data):
        score = data.get('score')
        player_name = data.get('player_name')

        # Perform the data validation.
        if not score:
            raise ValidationError({
                'score': 'This field is required.'
            })
        if not player_name:
            raise ValidationError({
                'player_name': 'This field is required.'
            })
        if len(player_name) > 10:
            raise ValidationError({
                'player_name': 'May not be more than 10 characters.'
            })

        # Return the validated values. This will be available as
        # the `.validated_data` property.
        return {
            'score': int(score),
            'player_name': player_name
        }

    def to_representation(self, obj):
        return {
            'score': obj.score,
            'player_name': obj.player_name
        }

    def create(self, validated_data):
        return HighScore.objects.create(**validated_data)
```

## Creating new base classes

`BaseSerializer`클래스는 특정 serializer 스타일을 처리하거나 대체 저장 백엔드와 통합하기 위해 새 generic serializer 클래스를 구현하려는 경우에도 유용하다.
다음 클래스는 임의의 객체를 기본 자료형으로 강제 변환할 수 있는 일반 serializer 컨버터의 예이다.

```python
class ObjectSerializer(serializers.BaseSerializer):
    """
    A read-only serializer that coerces arbitrary complex objects
    into primitive representations.
    """
    def to_representation(self, obj):
        for attribute_name in dir(obj):
            attribute = getattr(obj, attribute_name)
            if attribute_name('_'):
                # Ignore private attributes.
                pass
            elif hasattr(attribute, '__call__'):
                # Ignore methods and other callables.
                pass
            elif isinstance(attribute, (str, int, bool, float, type(None))):
                # Primitive types can be passed through unmodified.
                output[attribute_name] = attribute
            elif isinstance(attribute, list):
                # Recursively deal with items in lists.
                output[attribute_name] = [
                    self.to_representation(item) for item in attribute
                ]
            elif isinstance(attribute, dict):
                # Recursively deal with items in dictionaries.
                output[attribute_name] = {
                    str(key): self.to_representation(value)
                    for key, value in attribute.items()
                }
            else:
                # Force anything else to its string representation.
                output[attribute_name] = str(attribute)
```

# Advanced Serializer Usage

## Overriding serialization and deserialization behavior

serializer 클래스의 serialization, deserialization 또는 유효성 검사를 변경해야하는 경우 `.to_representation()` 또는 `.to_internal_value()` 메서드를 재정의해야한다.
유용한 이유는 다음과 같다.

- 새로운 serializer Base Class에 대한 새로운 동작을 추가
- 기존 클래스의 동작을 약간 수정
- 많은 양의 데이터를 반환하며 자주 액세스 되는 API 엔드포인트의 serializer 성능 향상

`to_representation(self, obj)`

serializer가 필요한 객체 인스턴스를 가져와서 표현을 반환해야한다. 일반적으로 이것은 내장 파이썬 데이터 유형의 구조를 반환하는 것을 의미한다. 처리 할 수 있는 정확한 유형은 API에 대해 구성한 렌더링 클래스에 따라 다르다.

`to_internal_value(self, data)`

검증되지 않은 데이터를 입력받아 `serializer.validated_data`로 사용할 수 있는 유효성 검사된 데이터를 반환해야한다. serializer 클래스에서 `.save()`가 호출되면 반환 값도 `.create()` 또는 `.update()`로 전달된다.
유효성 검사가 실패하면 메서드는 `serializers.ValidationError`를 발생시켜야한다. 일반적으로 여기에 있는 `errors`인수는 필드 이름을 오류 메세지에 매핑하는 dict이다.
이 메서드에 전달된 `data`인수는 일반적으로 `request.data`의 값이므로, 제공하는 데이터 유형은 API에 대해 구성한 파서 클래스에 따라 다르다.

## Serializer Inheritance

Django 폼과 마찬가지로 상속을 통해 serializer를 확장하고 다시 사용할 수 있다. 이를 통해 많은 수의 serializer에서 사용할 수 있는 부모 클래스의 공통 필드 또는 메서드 집합을 선언할 수 있다.

```python
class MyBaseSerializer(Serializer):
    my_field = serializers.CharField()

    def validate_my_field(self):
        ...

class MySerializer(MyBaseSerializer):
    ...
```

django의 `Model`과 `ModelForm`클래스처럼, serializer의 내부 `Meta`클래스는 부모의 내부 `Meta`클래스를 상속받지 않는다. `Meta`클래스가 부모 클래스에서 상속받기 원한다면 명시해야한다.


```python
class AccountSerializer(MyBaseSerializer):
    class Meta(MyBaseSerializer.Meta):
        model = Account
```

일반적으로 내부 메타 클래스에서는 상속을 사용하지 않고 모든 옵션을 명시적으로 선언하는 것이 좋다.
또한 다음과 같은 주의사항이 serializer 상속에 적용된다.

- 일반적인 Python 이름 해석 규칙이 적용된다. `Meta` 내부 클래스를 선언하는 여러 Base Class가 있는 경우, 첫번째 클래스만 사용된다. 이것은 자식의 메타가 존재한다면 자식의 메타를 의미하고, 그렇지 않으면 부모의 메타를 의미한다.
- 하위 클래스에서 이름을 없음으로 설정하여 부모 클래스에서 상속된 `field`를 선언으로 제거할 수 있다.


```python
class MyBaseSerializer(ModelSerializer):
    my_field = serializers.CharField()

class MySerializer(MyBaseSerializer):
    my_field = None
```

그러나 이 방법을 사용하는 경우에만 상위 클래스에 의해 선언적으로 정의 된 필드에서 선택 해제 할 수 있다. `ModelSerializer`가 디폴트 필드를 생성하는 것을 막지는 않는다. 기본 필드에서 선택 해제하려면 (Specifying which fields to include)를 참조하라.

## Dynamically modifying fields

serializer가 초기화되면 serializer에서 설정된 필드 dict에 `.fields` 특성을 사용하여 액세스할 수 있다. 이 속성에 액세스하고 수정하면 serializer 컨버터를 동적으로 수정할 수 있다.
`fields`인수를 직접 수정하면 serializer 선언 시점이 아닌 런타임시 serializer 필드의 인수 변경과 같은 흥미로운 작업을 수행할 수 있다.

### Example

예를 들어, serializer에서 초기화할 때 사용할 필드를 설정하려면 다음과 같이 serializer 클래스를 만들 수 있다.

```python
class DynamicFieldsModelSerializer(serializers.ModelSerializer):
    """
    A ModelSerializer that takes an additional `fields` argument that
    controls which fields should be displayed.
    """

    def __init__(self, *args, **kwargs):
        # Don't pass the 'fields' arg up to the superclass
        fields = kwargs.pop('fields', None)

        # Instantiate the superclass normally
        super(DynamicFieldsModelSerializer, self).__init__(*args, **kwargs)

        if fields is not None:
            # Drop any fields that are not specified in the `fields` argument.
            allowed = set(fields)
            existing = set(self.fields.keys())
            for field_name in existing - allowed:
                self.fields.pop(field_name)
```

이렇게 하면 다음을 수행할 수 있다.

```python
class UserSerializer(DynamicFieldsModelSerializer):
  class Meta:
    model = User
    fields = ('id', 'username', 'email')

print UserSerializer(user)
{'id': 2, 'username': 'jonwatts', 'email': 'jon@example.com'}

print UserSerializer(user, fields=('id', 'email'))
{'id': 2, 'email': 'jon@example.com'}
```

## Customizing The Default Fields

REST 프레임워크2에서는 개발자가 `ModelSeirlizer`클래스가 기본 필드 set을 자동으로 생성하는 방법을 재정의할 수 있는 API를 제공했다.
이 API에서는 `.get_field()`, `.get_pk_field()`와 그 외의 메소드가 포함되어있다.
하지만 serializer가 근본적으로 3.0으로 다시 디자인되었기 때문에 이 API는 더이상 존재하지 않습니다. 생성된 필드는 여전히 수정할 수 있지만 소스코드를 참조해야하며, 변경사항이 API의 비공개 비트에 해당하면 변경될 수 있음을 알고있어야한다.
