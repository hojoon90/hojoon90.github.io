---
layout: post
title: "ObjectMapper"
tags: [Spring]
comments: true
---

Json 데이터를 객체에 매핑할 때 많이 사용하는 객체가 바로 Jackson ObjectMapper다.  
ObjectMapper 를 사용하는 가장 큰 이유는 JSON 데이터를 객체로 손쉽게 매핑해주기 때문인데, JSON을 객체에 매핑하거나 객체를 JSON으로 변환할 때
일련의 과정을 거쳐 데이터 변환이 이루어 진다. 이 글에서는 사용 방법과 동작 원리에 대해 간단히 정리 해보려 한다.

## ObjectMapper 의존성 추가
의존성은 아래와 같은 방법으로 추가할 수 있다.

### Maven
pom.xml 안에 아래와 같이 추가.
```xml
  <dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>${jackson.version}</version>
  </dependency>
```
### Gradle
build.gradle 안에 아래와 같이 추가.
```groovy
    implementation 'com.fasterxml.jackson.core:jackson-databind:2.12.3'
```

## 사용법
간단한 예시로 확인해보자.
```java
@Getter
@NoArgsConstructor
public class User{
    private Long userId;
    private String accountId;
    private String name;
    private String email;
}
```
유저 정보를 제공해주는 API를 이용해 유저 객체를 만들어 보려 한다. 객체는 Getter와 빈 생성자만 만들도록 해주었다.  
제공되는 JSON 은 아래와 같다.
```json
{
  "userId": 21,
  "accountId": "sirlewis123",
  "userName": "Lewis Hamilton",
  "email": "sirlewis123@test.com",
  "phone": "010-0123-4567",
  "gender": "M"
}

```
위와 같이 API 를 통해 응답값이 오면 대개 String 형태로 반환이 된다. 반환된 응답값은 다음과 같이 객체로 변환해주면 된다.  
여기서는 API를 통해 json 데이터를 가져왔다고 가정하고 response값을 스태틱으로 만들었다.
```java
String response ="{\"userId\": 21,\"accountId\": \"sirlewis123\", \"userName\": \"Lewis Hamilton\", \"email\": \"sirlewis123@test.com\", \"phone\": \"010-0123-4567\", \"gender\": \"M\"}";
ObjectMapper objectMapper = new ObjectMapper()
        .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false); // 객체에 없는 변수 무시

//Json String 데이터를 객체로 변환
User user; 
try{
    //readValue는 JsonProcessingException, JsonMappingException 예외를 던지므로, 해당 예외의 최상위 예외인 IOException으로 잡아준다.
    user = objectMapper.readValue(content, User.class);
} catch (IOException ie){
    log.error("IOException", ie);
}
```
이렇게 하면 JSON String 으로 받아온 데이터를 객체로 손쉽게 변환할 수 있다.  
반대의 경우도 가능하다.
```java
@Setter
public static class Address{
    private int zipCode;
    private String address;
    private String addressDetail;
}

public static void main(String[] args){
    ObjectMapper objectMapper = new ObjectMapper();

    //Json String 데이터를 객체로 변환
    Address address = new Address();
    address.setZipCode(12345);
    address.setAddress("서울특별시 중구 세종대로 110");
    address.setAddressDetail("서울시청");
    
    try{
        //readValue는 JsonProcessingException, JsonMappingException 예외를 던지므로, 해당 예외의 최상위 예외인 IOException으로 잡아준다.
        String jsonData = objectMapper.writeValueAsString(address);
    } catch (IOException ie){
        log.error("IOException", ie);
    }
}

```
이렇게 하면 Address 객체는 다음과 같은 String 형태의 JSON 데이터가 된다.
```json
{
  "zipCode": 12345,
  "address": "서울특별시 중구 세종대로 110",
  "addressDetail": "서울 시청"
} 
```
위와 같이 JSON -> 객체로 변환하는 것을 역직렬화(Deserialize), 객체 -> JSON 으로 변환시키는 것을 직렬화(Serialize)라고 한다.

objectMapper 객체를 생성할 때 설정을 하나 넣어주었는데, 역직렬화 시 JSON에는 존재하지만 객체에 없는 경우 해당 필드를 무시하는 옵션을 같이 추가해주었다.  
이렇게 해주지 않으면 역직렬화 과정에서 ObjectMapper가 존재하지 않는 값을 처리하려다가 예외를 발생시키게 된다.  
여기서 한가지 신기한 점은 역직렬화 시 객체에 setter나 변수를 받는 생성자가 존재하지 않는데, 객체에 알아서 매핑되어 생성된다는 것이다.  
ObjectMapper 가 동작하는 원리는 아래와 같다.

## 동작 원리
ObjectMapper의 readValue 기준으로 코드 내부를 보면 다음과 같다.
```java
...

public <T> T readValue(String content, Class<T> valueType) throws JsonProcessingException, JsonMappingException {
    this._assertNotNull("content", content);
    return this.readValue(content, this._typeFactory.constructType(valueType));
}

...

public <T> T readValue(String content, JavaType valueType) throws JsonProcessingException, JsonMappingException {
    this._assertNotNull("content", content);

    try {
        return this._readMapAndClose(this._jsonFactory.createParser(content), valueType);
    } catch (JsonProcessingException var4) {
        throw var4;
    } catch (IOException var5) {
        throw JsonMappingException.fromUnexpectedIOE(var5);
    }
}
```
readValue 메서드는 다양한 데이터들을 처리하기 위해 여러개가 오버로딩 되어있다.   
일단 코드를 보면 결국 데이터 반환을 위해선 실제 데이터의 String 값과 JavaType을 받는것을 알 수 있다.  
또한 return 쪽의 this._jsonFactory.createParser를 통해 데이터를 JsonParser형태로 만들어주는 것도 볼 수 있다.
```java
 protected Object _readMapAndClose(JsonParser p0, JavaType valueType) throws IOException {
        JsonParser p = p0;
        Throwable var4 = null;

        Object var9;
        try {
            DeserializationConfig cfg = this.getDeserializationConfig();
            DefaultDeserializationContext ctxt = this.createDeserializationContext(p, cfg);
            JsonToken t = this._initForReading(p, valueType);
            Object result;
            if (t == JsonToken.VALUE_NULL) {
                result = this._findRootDeserializer(ctxt, valueType).getNullValue(ctxt);
            } else if (t != JsonToken.END_ARRAY && t != JsonToken.END_OBJECT) {
                result = ctxt.readRootValue(p, valueType, this._findRootDeserializer(ctxt, valueType), (Object)null);
                ctxt.checkUnresolvedObjectId();
            } else {
                result = null;
            }

            ...
```
_readMapAndClose 메서드에서 핵심적인 부분은 바로 else if 부분의 로직이다.  
변환한 JsonToken값의 끝나는 값이 배열형태([]) 혹은 객체 형태({})일 경우 실제 데이터를 객체로 변환해주는 작업을 처리한다.  
this._findRootDeserializer 메서드를 파고 들어가보면 ctxt (DefaultDeserializationContext)객체에 valueType을 던져서 알맞은 Deserialize 객체를 찾아오는데,
내부로직은 따로 여기서는 설명하지 않도록 하겠다.
```java
public Object readRootValue(JsonParser p, JavaType valueType, JsonDeserializer<Object> deser, Object valueToUpdate) throws IOException {
    if (this._config.useRootWrapping()) {
        return this._unwrapAndDeserialize(p, valueType, deser, valueToUpdate);
    } else {
        return valueToUpdate == null ? deser.deserialize(p, this) : deser.deserialize(p, this, valueToUpdate);
    }
}
```
readRootValue 객체에서는 실제 값을 deserialize 처리해주는 메서드를 호출해준다. ObjectMapper 설정값에 useRootWrapping 설정을 별도로 안해주었으면
else 부분을 확인하면 된다.  
파라미터 중에 JsonDeserializer<Object> deser 가 존재하는데, 위의 메서드에서 찾았던 Deserialize 객체가 넘어오게 된다.  
JsonDeserializer는 추상객체로서 해당 객체를 상속 받는 여러 객체가 있는데, 예시의 경우에선 BeanDeserializer 객체를 반환받게 된다.

```java
public Object deserialize(JsonParser p, DeserializationContext ctxt) throws IOException {
    if (p.isExpectedStartObjectToken()) {
        if (this._vanillaProcessing) {
            return this.vanillaDeserialize(p, ctxt, p.nextToken());
        } else {
            p.nextToken();
            return this._objectIdReader != null ? this.deserializeWithObjectId(p, ctxt) : this.deserializeFromObject(p, ctxt);
        }
    } else {
        return this._deserializeOther(p, ctxt, p.currentToken());
    }
}

...

private final Object vanillaDeserialize(JsonParser p, DeserializationContext ctxt, JsonToken t) throws IOException {
    Object bean = this._valueInstantiator.createUsingDefault(ctxt);
    p.setCurrentValue(bean);
    if (p.hasTokenId(5)) {
        String propName = p.currentName();

        do {
            p.nextToken();
            SettableBeanProperty prop = this._beanProperties.find(propName);
            if (prop != null) {
                try {
                    prop.deserializeAndSet(p, ctxt, bean);
                } catch (Exception var8) {
                    this.wrapAndThrow(var8, bean, propName, ctxt);
                }
            } else {
                this.handleUnknownVanilla(p, ctxt, bean, propName);
            }
        } while((propName = p.nextFieldName()) != null);
    }

    return bean;
}
```
BeanDeserializer 객체의 일부 코드이다. 이제 앞 메서드에서 넘어온 데이터들은 vanillaDeserialize 메서드를 타게 되고, 여기서 실제 객체 매핑이 일어나게 된다.  
vanillaDeserialize 메서드 맨 첫줄의 createUsingDefault 메서드에서 아무 변수가 없는 빈 생성자를 호출하여 객체를 생성한다.  
그리고 while문을 통해 실제 객체필드에 json으로 넘어왔던 값들을 세팅해주는 로직이 동작하게 된다(prop.deserializeAndSet. prop 변수는 FieldProperty 객체이다.).  
그러고 나서 bean 변수를 return 하면서 JSON 데이터를 객체로 매핑하게 된다.

결국 내부 로직에서 빈 생성자를 만들고, 해당 키를 찾아서 빈 객체에다가 값을 내부적으로 세팅해주기 때문에 우리가 만든 객체에서는
별도의 setter나 모든 파라미터를 받는 생성자가 필요없는 것이다.

## 참고 사이트
https://interconnection.tistory.com/137  
https://www.baeldung.com/jackson-object-mapper-tutorial
https://sedangdang.tistory.com/307