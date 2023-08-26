# @Transactional과 쓰기지연

## 현상
배치의 시작과 종료에 대한 상태를 DB에 실시간으로 저장할 일이 생겨, 상태를 실시간으로 저장하려 했으나 
모든 배치 작업이 완료된 후 DB에 데이터가 Insert 되어 상태가 실시간으로 DB에 저장되지 않음.

## 원인
결과적으로 JPA 환경에서 `@Transactional` 로 인한 쓰기 지연이 동작하게 되여 DB에 실시간으로 저장되지 않음.\
실시간으로 저장할 로직에 `@Transactional` 을 제거하여 쓰기지연 없이 실시간 상태 저장하도록 변경.\
쓰기 지연에 대해 정확하게 알지 못한 채로 사용.

### 상세
선언적트랜잭션(`@Transactional`)이 달려있는 메소드들은 하나의 작업단위로 묶이게 된다.\
JPA는 `@Transactional` 이 메소드에 달려있을 경우 쓰기 지연이 동작하게 된다. 쓰기 지연은 다음과 같이 동작한다.

1. `@Transactional` 이 있는 메소드 안에서 save(Insert 쿼리) 호출.
2. Insert SQL 이 생성되지만 바로 실행되지 않고 쓰기 지연 SQL 저장소에 계속 저장 됨.
3. 트랜잭션이 commit 됨.
4. 엔티티 매니저가 영속성 컨텍스트를 flush 시키고, 이때 저장 되었던 쿼리 들이 모두 DB에 반영함.

기존에 적용되었던 코드는 아래와 같다.

```java

@Transactional
public void setBatchLog(Long Id, String type, int status, String message){
    LogBatch batch = //로그 생성
    logBatchRepository.save(batch);
}

...

public void process(Store store) {
    try{
        if (store.getHostingSite() == HostingType.A) {
            AsiteBatchManager(store);
        } else {
            BsiteBatchManager(store);
        }
    }catch (Exception e){
        setBatchLog(store.getId(), "ITEM", 2, "Item Refresh Fail... Store Id is: "+store.getId());
        throw new BusinessException(UPDATE_ITEM_ERROR);
    }
}

...

@Transactional
public void updateItemDataManual(Long id) {
    setBatchLog(id, "ITEM", 0, "Item Refresh Processing...");

    Store store = storeRepository.findById(storeId).orElseThrow(() -> new BusinessException(STORE_NOT_FOUND));
    process(store);

}

...
```

- updateItemDataManual(Item 수동 업데이트),  setBatchLog(배치 상태 저장) 메소드에 `@Transactional` 이 적용되어 있음.
- updateItemDataManual은 실제 Item 데이터가 업데이트되는 메소드인 process 메소드를 실행시킴.
- 이렇게 될 경우 배치 상태 저장과 Item 수동 업데이트가 하나의 트랜잭션으로 묶이게 됨.
- 하나의 트랜잭션으로 묶였기 때문에 JPA의 쓰기지연이 동작.
- 결국 배치 상태 저장 역시 실시간이 아닌 쓰기 지연 저장소에 저장된 후, 커밋 시점에 모두 DB에 저장되어 버림.

원하는 동작은 배치 상태는 실시간으로 저장되길 원하므로 아래와 같이 코드 변경.

```java

public void setBatchLog(Long Id, String type, int status, String message){
    LogBatch batch = //로그 생성
    logBatchRepository.save(batch);
}

...

@Transactional
public void process(Store store) {
    try{
        if (store.getHostingSite() == HostingType.A) {
            AsiteBatchManager(store);
        } else {
            BsiteBatchManager(store);
        }
    }catch (Exception e){
        setBatchLog(store.getId(), "ITEM", 2, "Item Refresh Fail... Store Id is: "+store.getId());
        throw new BusinessException(UPDATE_ITEM_ERROR);
    }
}

...

public void updateItemDataManual(Long id) {
    setBatchLog(id, "ITEM", 0, "Item Refresh Processing...");

    Store store = storeRepository.findById(storeId).orElseThrow(() -> new BusinessException(STORE_NOT_FOUND));
    process(store);
}

...
```
- 실제 Item을 업데이트 하는 process 메소드에 `@Transactional`을 적용
- updateItemDataManual, setBatchLog는 `@Transactional`이 제거되었으므로 하나의 트랜잭션으로 묶이지 않게 됨.
- setBatchLog 메소드는 더이상 쓰기 지연의 영향을 받지 않게 되고, 실시간으로 DB에 로그를 저장할 수 있게 됨.