---
layout: post
title: "MapStruct 를 통한 매핑처리"
tags: [Java, Spring]
comments: true
---

사이드 프로젝트를 진행하면서 MapStruct를 이용하여 데이터를 변환하는 방식을 사용해보았다.  
소감은 생각보다 데이터 변환에 있어 편리한 점이 있고, 필드가 많아질 때 일일히 다 작성해주지 않아도 되는 점이 마음에 들었다.  
반면 러닝커브가 크진 않지만 조금은 배우고 써야 하는점이 있었으며, 인터페이스를 통해 데이터를 변환해주다 보니,
추후 유지보수를 진행할 때 괜찮을까..? 싶은 점이 몇가지 있긴 했다.  
여기서는 기본적인 사용 방법에 대해 작성해본다.

## 라이브러리 추가
그래들 기준으로 아래와 같이 라이브러리를 추가해준다.
```groovy
dependencies {
    ...
    implementation 'org.mapstruct:mapstruct:1.5.5.Final'
    annotationProcessor 'org.mapstruct:mapstruct-processor:1.5.5.Final'
}
```
라이브러리 추가가 완료되면 다음과 같이 Mapper를 만들어준다.

## Mapper
```java
@Mapper(
        componentModel = "spring",
        injectionStrategy = InjectionStrategy.CONSTRUCTOR,
        unmappedTargetPolicy = ReportingPolicy.ERROR,
        uses = {
                EngineMapper.class,
                WheelMapper.class
        }
)
public interface CarMapper{
    
//    DI 프레임워크 미사용 시 관례적으로 INSTANCE를 명시해준다.
//    public static final CarMapper INSTANCE = Mappers.getMapper( CarMapper.class );

    @Mapping(source = "front", target = "bonnet")
    @Mapping(source = "rear", target = "trunk")
    CarDto convertDto(Car entity); 
}

```
코드들을 설명하면 다음과 같다.

### @Mapper
해당 인터페이스가 Mapper라는 것을 명시해준다. 이 애노테이션은 몇가지 옵션을 추가해줄 수 있다. 여기에 추가된 옵션은 아래와같다.
* componentModel:
    * 이 매퍼를 어떤 모델로 구성하여 만들지 정해준다. 위와 같이 'spring' 이라고 명시해 줄 경우 해당 매퍼는 스프링 빈으로 등록 되며 @Autowired 를 통해 검색 된다.
* injectionStrategy:
    * 종속성 주입 방식을 선택한다. 'field'(필드), 'constructor'(생성자) 두가지 방식이 있으며, 애노테이션 기반에서만 사용된다 (Spring, CDI, JSR330 등)
    * 생성된 매퍼는 MapStruct가 매핑을 위해 인스턴스를 사용해야 한다는 것을 감지한 경우 'uses' 속성에 정의된 클래스를 넣어준다. 위와 같은 경우는 EngineMapper, WheelMapper를 넣어준다.
* unmappedTargetPolicy:
    * 매핑되지 않는 변수들이 있을 경우 처리되는 방식을 정한다. 'ERROR', 'WARN', 'IGNORE' 방식이 있다.
* uses:
    * 이 매퍼에 같이 사용되는 매퍼를 명시한다.

### INSTANCE
DI 프레임 워크를 사용하지 않으면 관례적으로 매퍼 생성시엔 INSTANCE를 상수로 생성해주어야 한다.
그래야 새 인스턴스를 계속 생성하지 않고 하나의 인스턴스로 관리를 해줄 수 있다.   
반면 DI 프레임워크(Spring, CDI 등)를 사용할 경우 Mapper 클래스를 생성하지 않고 종속성 주입을 통해 사용하는 것을 더 추천한다.
이미 MapStruct 쪽에서도 해당 방식을 지원하고 있기 때문에 굳이 인스턴스를 만들 필요는 없다. 대신 위 옵션 처럼 componentModel 옵션에
사용하는 모델에 대한 명시를 해주면 된다.

### @Mapping
변수로 들어오는 객체와 변환할 객체의 필드가 다를 경우 수동으로 필드를 매핑해줄 때 사용한다. source의 경우 파라미터에 들어오는 객체의 필드를 넣어주고
target은 변환할 객체의 필드를 넣어주면 된다.
### convertDto
실제 데이터를 변환해주는 메서드이다. 위 메서드를 설명하면 'Car 엔티티를 CarDto로 변환'해주는 것이다. 이 부분은 프로젝트 빌드 시
구현체가 자동으로 생성된다. 아래와 같은 형태로 만들어진다.

## 구현체
```java
@Getter
@Builder
public class Car{
    
    private String door;
    private String front;
    private String rear;
    private Engine engine;
    private Wheel wheel;
}
```

```java
public class CarDto{

  private String door;
  private String bonnet;
  private String trunk;
  private Engine engine;
  private Wheel wheel;
}
```

```java
@Generated(
    value = "org.mapstruct.ap.MappingProcessor",
    date = "2024-03-08T15:14:19+0900",
    comments = "version: 1.4.2.Final, compiler: javac, environment: Java 17.0.1 (Azul Systems, Inc.)"
)
@Component
public class CarMapperImpl implements CarMapper {

    private final EngineMapper engineMapper;
    private final WheelMapper wheelMapper;

    @Autowired
    public CarMapperImpl(EngineMapper engineMapper, WheelMapper wheelMapper) {

        this.engineMapper = engineMapper;
        this.wheelMapper = wheelMapper;
    }

    @Override
    public com.carfactory.car.domain.dto.CarDto convertDto(Car entity) {
        if ( entity == null ) {
            return null;
        }

        CarDtoBuilder<?, ?> carDto = com.carfactory.car.domain.dto.CarDto.builder();

        carDto.door( entity.getDoor() );
        carDto.bonnet( entity.getFront() );
        carDto.trunk( entity.getRear() );
        carDto.engine( engineMapper.of( entity.getCarEngine() ) );
        carDto.wheel( wheelMapper.of( entity.getTire() ) );

        return carDto.build();
    }
}
```
위 클래스는 MapStruct가 자동으로 생성해준 구현체 클래스이다. 예시로 만든 클래스이지만, 대략 이런 식으로 구현체를 알아서 생성해준다.
사용자는 인터페이스만을 사용하여 데이터 변환을 처리하면, 실제 구현된 구현체를 통해 데이터 변환이 이루어지는 것이다.
구현체는 특별하게 설명할 부분이 없으므로 바로 넘어가도록 하겠다.

### 실제 사용 코드
만들어진 매퍼는 아래와 같이 사용할 수 있다.
```java
public class Temp{
  ...
  private final CarMapper carMapper;
  ...
  
  CarDto dto = carMapper.convertDto(car);
}

```
단순하게 매퍼를 빈으로 주입한 후 메서드하나만 호출해주면, 알아서 변환된 Dto 값을 반환해준다. 이는 개발자가 작성해야 할 코드를 획기적으로 많이 줄여주게 된다.
특히 매핑해주어야할 필드의 개수가 많을 경우, 이 방식은 더욱 개발 시 효율적으로 사용할 수 있게 된다.

## 마무리
MapStruct 사용에 대해 이번에 간단하게 정리해보았다. MapStruct의 사용에 대해 현재는 의견이 좀 나뉘고 있는 부분이 있다.  
그 중 하나가 매퍼의 빈 등록이다. 아래 참고 블로그에 잘 작성되어 있지만, 스프링에서 사용할 경우 매퍼를 빈에 등록하여 사용하게 된다.
이는 곧 스프링에서 빈을 직접 관리하게 된다는 이야기이다. 이렇게 될 경우 의존 관계가 생성되고, 테스트 코드 작성 시 상당히 곤란해지게 된다.
실제 feign도 그렇고 테스트 코드 작성 시에 빈 의존성으로 인해 테스트하기가 꽤나 까다롭다...   
그러나 그 부분들을 감안하더라도 개발자가 작성할 코드가 현저히 줄어들고, 이로 인한 개발 시간 단축을 위해서라면 한번쯤 사용해봐도 괜찮다고 생각한다.

## 참고
https://mapstruct.org/
https://mapstruct.org/documentation/dev/reference/html/
https://marklee1117.tistory.com/121
https://kth990303.tistory.com/403

