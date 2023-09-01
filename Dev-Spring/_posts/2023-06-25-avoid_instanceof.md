# instanceof 를 지양하자

* instanceof → 클래스 타입을 체크하는 연산자
* instanceof 연산자는 주로 클래스들이 상속 관계일 때 사용 가능.
* EP 처리 시 변환시킬 상품 정보의 타입을 체크하여 처리

```java
private Item convertProductData(Long storeId, Object productData){

    Item item = null;

    if (productData instanceof AProduct) {
                    ...
    } else if (productData instanceof BProduct){
        ...
    }

    return item;
}
```

* 하지만 이러한 instanceof 처리는 다음과 같은 문제가 있음.
1. 추상화계층을 깨트림.
    * 외부에서 추상화된 상위 계층을 사용하는 입장에서는 하위계층을 알아서도, 알 필요도 없음.
    * 위 코드에서는 서비스 로직안에 있는 convertProductData 메소드가 productData의 타입을 알 필요가 없음. (추상화)
    * 또한 하위 계층의 로직을 알 필요도 없음 (캡슐화)
    * 단순히 productData를 Item 으로 바꿔줘! 라는 요청만 하면 됨.
2. OCP 위반
    * productData 타입이 추가될 때 마다 instanceof 추가가 필요.
    * productData의 타입을 다른곳에서도 체크한다면 그곳에도 추가되는 타입 세팅 필요.

#### 코드 리팩토링

* Object → ApiItemData 로 변경
* 각 클래스의 타입을 세팅.

```java
public abstract class ApiItemData {
    public abstract String getType();
}
```

```java
@Data
@JsonIgnoreProperties(ignoreUnknown = true)
public static class AProduct extends ApiItemData  {
        ...
    @Override
    public String getType() {
        return ExternalConstants.ITEM_TYPE_A;
    }
}
```

```java
@Data
@JsonIgnoreProperties(ignoreUnknown=true)
public class BProduct extends ApiItemData {
        ...
    @Override
    public String getType() {
        return ExternalConstants.ITEM_TYPE_B;
    }

}
```

* 서비스 로직 변경
```java
private Optional<Item> convertProductData(Long storeId, ApiItemData productData) {
    return itemConvertFactory.getApiItemData(storeId, productData);
}
```


* ProductData → Item 변경을 위한 클래스 분리
```java
@Component
@RequiredArgsConstructor
public class ItemConvertFactory {

    private final ItemCheckApiComponent itemCheckApiComponent;

    //여기서 받아온 상품정보 처리 후 Item으로 리턴
    public Optional<Item> getApiItemData(Long storeId, ApiItemData apiItemData){
        HostingType type = HostingType.valueOf(apiItemData.getType());
        ItemConvert converter = switch (type){
            case A -> new AItemConvert();
            case B -> new BItemConvert();
        };
        return converter.convert(itemCheckApiComponent, storeId, apiItemData);
    }
}
```

* 추상화 클래스와 구현 클래스
```java
public interface ItemConvert {
    Optional<Item> convert(ItemCheckApiComponent itemCheckApiComponent, Long storeId, ApiItemData itemData);
}
```

```java
public class AItemConvert implements ItemConvert{

    @Override
    public Optional<Item> convert(ItemCheckApiComponent itemCheckApiComponent, Long storeId, ApiItemData itemData) {
        AProduct data = new ObjectMapper().convertValue(itemData, AProduct.class);

        return Optional.ofNullable(
                Item.builder()
                        ...
                        .build()
        );
    }
        ...
}
```

```java
public class BItemConvert implements ItemConvert {

    @Override
    public Optional<Item> convert(ItemCheckApiComponent itemCheckApiComponent, Long storeId, ApiItemData itemData) {
        BProduct data = new ObjectMapper().convertValue(itemData, BProduct.class);

          ...

        return Optional.ofNullable(
                Item.builder()
                        ...
                        .build()
        );
    }
        ...
}
```