# Authentication

인증은 view에서 인증이 실행되기 전에 실행된다.
`request.auth`의 추가적 인증 정보를 사용하게 될것이다.

인증체계는 항상 클래스 목록으로 정의된다. (Facebook API, 그냥 로그인된 사용자)

기본적으로 Basic, Sessions를 가지고 있다.
```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        # IOS에도 사용가능하므로 위의 것을 사용하는 것이 낫다.
        'rest_framework.authentication.BasicAuthentication',
        # IOS에는 사용되지 않는 장고 쿠키 세션 기반, Front, HTML 에서 사용함
        # Browerseralbe 하게 확인 가능, admin 로그인하고 API 요청해보면 로그인되있는걸 확인 가능
        'rest_framework.authentication.SessionAuthentication',
    )
}
```

`BasicAuth`같은 경 Base64 인코딩 방식을 사용하여 ID/PW쌍의 인증 정보를 전달한다.
이 인증스키마는 복호화가 가능하므로 안전하지 않다.

대신 Token Authentication 방법을 사용한다.

`INSTALLED_APPS`에 `rest_framework.authtoken`을 추가 후 migrate

```
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        # 테스트시에 간단하다.
        'rest_framework.authentication.BasicAuthentication',
        # 테스트시에는 많이 귀찮다.
        'rest_framework.authentication.TokenAuthentication',
    ),
    # ...
```

## User 인증 토큰 받아오는 API

```python
class AuthTokenView(APIView):
    # URL: '/members/auth-token/'
    def post(self, request):
        # username, password를 받음
        # 받은 내용으로부터 authenticate를 실행
        # - 인증 성공시
        #       인증된 User에 연결되는 Token을 가져오거나 생성(get_or_create)
        #       Token의 Key값을 Response에 답아 돌려줌
        # - 인증 실패시
        #       AuthenticationFailed Exception을 raise
        username = request.POST.get('username')
        password = request.POST.get('password')

        user = authenticate(username=username, password=password)

        if user is not None:
            token, created = Token.objects.get_or_create(user=user)
            return Response({'Token': token.key})
        else:
            raise AuthenticationFailed
```

`AuthTokenView` view로 Header에 `username, password`를 보내준다. `multipart/form-data`로 데이터를 전송하기 위해서 Parser가 필요하다.

```python
'DEFAULT_PARSER_CLASSES': (
    'rest_framework.parsers.JSONParser',
    'rest_framework_yaml.parsers.YAMLParser',
    'rest_framework_xml.parsers.XMLParser',
    'rest_framework.parsers.FormParser',
    'rest_framework.parsers.MultiPartParser'
),
```

이후 재전송하면 정상적으로 데이터를 파싱할 수 있다.

## 인증된 User의 정보를 가져오는 API

```python
class ProfileView(APIView):
    # URL: '/members/profile/'
    # 인증된 사용자만 접근 가능하도록 permission_classes 설정
    #   위의 AuthTokenView에서 전달받은 Token key 값을
    #   Postman의 HTTP 요청 헤더에
    #   Authorization: Token <Key값> 으로 세팅해서 유저 인증이 성공하는지 확인
    permission_classes = (IsAuthenticated, )

    def get(self, request):
        # 현재 요청에 해당하는 User의 정보를
        # UserSerializer로 직렬화 한 후 Response에 보내줌
        serializer = UserDetailSerializer(request.user)
        # `data=` 로 넣어주지 않은 Serialzer 객체는 .is_valid() 속성이 존재하지 않는다.

        return Response(serializer.data)
```

`ProfileView`에 `Authorization` 필드에 `Token <위에서 받은 토큰>`을 작성하여 보내주면 자동으로 request.user에 Token값이 같은 유저를 넣는다. 그렇기때문에 request.user를 사용할 수 있다.
