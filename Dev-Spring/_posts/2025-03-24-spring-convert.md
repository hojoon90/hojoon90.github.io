---
layout: post
title: "Converter 인터페이스"
tags: [Spring]
comments: true
---

## 문제점

프로젝트 개발을 진행하다 보면 타입 변경 케이스가 많은데 대부분 아래와 같이 작성되어 있다.
```java
Integer statsDt = Integer.parseInt(j.getCreatedAt().format(DateTimeFormatter.ofPattern(ShopConstants.FORMAT_DATE)));
```
위 코드는 날짜 객체를 Integer로 바꾸는 코드인데, 타입 변환을 위해 포맷, 패턴등을 바꾸다 보니 코드가 길어지게 되었다. 타입 변환이 필요할 때마다 이렇게 작성하게 되면 코드 작성량이 늘어나게 된다.  
또 하나 문제점은 이런 코드가 타입을 변경하는곳마다 존재하고 있으며, 공통으로 관리되고 있지 않다.
```java
userStoreList.stream().forEach(j -> {  
	Integer statsDt = Integer.parseInt(j.getCreatedAt().format(DateTimeFormatter.ofPattern(ShopConstants.FORMAT_DATE)));  
	...
});

public List<CountByDateDto> getVisitCount(Period period) {  
	int startDate = Integer.parseInt(period.getStartDate().format(DateTimeFormatter.ofPattern(ShopConstants.FORMAT_DATE)));
	...
}

//매 시간 5분마다 아이템 업데이트  
@Scheduled(cron = "0 5 */1 * * *")  
public void updateItemClickCount() {  
	Integer today = Integer.parseInt(LocalDate.now().format(DateTimeFormatter.ofPattern(ShopConstants.FORMAT_DATE)));
	...
}
```

타입 변환 로직을 공통적으로 사용하지 않을 경우, 수정사항이 생기면 일일히 찾아서 수정해주어야 한다.  
->  유지보수성이 나빠지게 된다.

## 코드 수정
타입 변경을 공통으로 처리할 수 있도록 스프링의 Converter라는 인터페이스를 활용하기로 함.  
### Converter 인터페이스
Converter는 스프링에서 제공하는 타입 변환기 인터페이스이다.
스프링에서 기본적으로 구현하여 제공해주는 컨버터들도 있으며, 사용자가 직접 해당 인터페이스를 상속받아 커스텀 컨버터를 구현할 수도 있다.
날짜를 숫자로 변환하는 컨버터는 따로 존재하지 않아 아래와 같이 커스텀으로 구현하였다.
```java
public class LocalDateTimeToIntegerConverter implements Converter<LocalDateTime, Integer> {  
	@Override  
	public Integer convert(LocalDateTime source) {  
		return Integer.valueOf(source.format(DateTimeFormatter.ofPattern(ShopConstants.FORMAT_DATE)));  
	}  
}
```

### 설정 방법
컨버터 인터페이스에 들어가보면 다음과 같이 정의되어있다.
```java
@FunctionalInterface  
public interface Converter<S, T> {  
  
	/**  
	* Convert the source object of type {@code S} to target type {@code T}.  
	* @param source the source object to convert, which must be an instance of {@code S} (never {@code null})  
	* @return the converted object, which must be an instance of {@code T} (potentially {@code null})  
	* @throws IllegalArgumentException if the source cannot be converted to the desired target type  
	*/  
	@Nullable  
	T convert(S source);
...
```

T 와 S 제네릭 타입이 있는데 T는 변환된 결과 타입, S는 변환시킬 타입을 입력해주면 된다. 실제 구현은 convert 메소드를 구현하여 값을 리턴해주면 된다.

컨버터 구현체를 빈 객체로 사용하기 위해선 먼저 컨버터를 빈으로 등록해주어야 한다.
방법은 2가지가 있는데, 아래와 같다
* Configure 설정 시 등록
* 자동으로 빈 등록

먼저 Configure 설정은 프로젝트가 Spring MVC 일때 많이 사용하는 방식이다.
```java
@Configuration  
public class WebConfig implements WebMvcConfigurer {  
	@Override  
	public void addFormatters(FormatterRegistry registry) {  
		registry.addConverter(new LocalDateTimeToIntegerConverter());  
	}
}
```
1. @Configuration으로 설정 빈을 만들어주고, WebMvcConfigurer 인터페이스를 구현할 수 있도록 한다.
2. addFormatters 메서드를 오버라이드 한다.
3. FormatterRegistry.addConverter를 통해 새로 만든 컨버터를 등록해준다.

FormatterRegistry는 Formatter를 등록해주는 인터페이스이지만, ConverterRegistry를 상속받고 있으므로 컨버터 등록 역시 가능하다.

Spring Boot 를 사용한다면 아래 방법으로 바로 등록하여 사용 가능하다.
```java
@Component
public class LocalDateTimeToIntegerConverter implements Converter<LocalDateTime, Integer> {  
	@Override  
	public Integer convert(LocalDateTime source) {  
		return Integer.valueOf(source.format(DateTimeFormatter.ofPattern(ShopConstants.FORMAT_DATE)));  
	}  
}
```
1. 컨버터 인터페이스를 구현하여 클래스를 만든다.
2. @Component 애노테이션으로 빈으로 등록해준다.

@Component 로 커스텀 컨버터를 등록 해주면 스프링 부트에서 자동으로 컨버터를 감지하여 FormattingConversionService에 만든 컨버터를 추가해준다.

### 실제 사용
실제 컨버터를 사용하는법은 간단하다.
우선 빈으로 등록된 컨버전 서비스를 사용할 서비스에 주입시켜준다.
```java
@Service  
@RequiredArgsConstructor  
@Slf4j  
public class WebAppUserManageService {  
	...
	private final ConversionService conversionService;
	...
}
```
또는
```java
@Service  
@Slf4j  
public class WebAppUserManageService {  
	...
	private final ConversionService conversionService;
	...
	public WebAppUserManageService (ConversionService conversionService){
		this.conversionService = conversionService;
	}
}

```

그리고 아래와 같이 타입 변환할 값을 conversionService 를 이용하여 변경해준다.
```java
//Integer statsDt = Integer.parseInt(j.getCreatedAt().format(DateTimeFormatter.ofPattern(ShopConstants.FORMAT_DATE)));
Integer statsDt = conversionService.convert(j.getCreatedAt(), Integer.class);
```
convert 메소드의 인자는 첫번째에 실제 변경할 대상을 넣어주고, 두번째엔 변경할 타입 클래스를 넣어주면 된다.
코드 양이 많이 줄어들었으며, 커스텀 컨버터 클래스에서 타입 변환을 해주므로 유지보수성도 높아지게 되었다.

## 결론
스프링에서 제공해주는 Converter 인터페이스와 ConversionService 사용을 통해 코드를 좀더 간결하게 변경할 수 있었으며, 유지보수성이 높은 코드로 변경할 수 있었다. 