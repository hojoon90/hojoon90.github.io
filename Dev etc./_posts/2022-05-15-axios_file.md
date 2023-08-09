---
layout: post
title: "[JavaScript] Axios를 이용하여 파일 업로드하기"
tags: [JavaScript, Axios]
comments: true
---

이번에 개발을 진행하면서, multifile 업로드 관련 로직을 처음 개발해보았다. 여러개의 이미지 파일을 서버쪽에 업로드해주는 로직인데,
파일 업로드는 처음이라 어떻게 업로드하고 어떤 삽질을 했는지 나중에 잊어먹지 않기 위해 작성해보려 한다.  
여담이지만 프론트쪽 개발은 현재 Vue3로 개발중인데 백엔드와 달리 디버깅하기가 너무 힘들어 삽질 시간이 한 2배는 더 걸리는 것 같다...

### 파일 업로드 코드

파일을 업로드 하기 위해 먼저 파일을 올리는 코드를 만들어준다.
```html
<template>
  <div class="room-file-upload-wrapper">
    <!-- 이미지 등록 input -->
    <div class="room-file-upload-example-container">
      <div class="imag-box">
        <label :for="'file'+idNo">
          <div class="file-textbox">
            <img src="../img/folder_img.png" alt="">
            <p>
              클릭해서 이미지 추가
            </p>
          </div>
        </label>
        <input type="file" :id="'file'+idNo" ref="files" @change="imageUpload" />
      </div>
    </div>
  </div>
</template>
```
```javascript
export default {
...
  methods: {
    imageUpload() {
      let num = -1;
      for (let i = 0; i < this.$refs.files.files.length; i++) {
        this.files = [
          ...this.files,
          {
            file: this.$refs.files.files[i],
            preview: URL.createObjectURL(this.$refs.files.files[i]),
            number: i,
            index: this.index,
          },
        ];
        num = i;
      }
      this.uploadImageIndex = num + 1;
    },
  }
};
```
특정 영역을 클릭하여 로컬에 있는 파일을 업로드하는 코드이다. html 부분의 input 태그에서 type을 **file**로 해주어야 파일 업로드가 가능하다.
파일이 업로드 되면 @change를 통해 imageUpload 메소드가 실행되면서, 서버에 업로드 할 수 있는 데이터 형태로 만들어준다.

### file 데이터 전송을 위한 formData 사용

다음은 위에 만든 file 데이터를 서버에 전송할 수 있도록 formData를 사용해준다. formData를 사용하여 파일 및 기타 정보들을 서버로 전달하는데,
맨날 Json만 만들어서 데이터 전송을 해보다 보니 처음에는 Json으로 보내려다가 계속 실패했었다...

일단 FormData 는 key-value로 구성되어 있다. 전달하고 싶은 데이터를 넣기 위해선

>formData.append('key', value)

형태로 넣어주면 된다.
이미지와 같은 파일을 보내기 위해선 이 형태로 데이터를 보내면 된다고 한다.
코드는 아래와 같이 작성해주면 된다.

```javascript
async sendFile(){
    ...

    let formData = new FormData(); // formData 객체를 생성한다.

    formData.append("file", item.file)
    
    formData.append("userId", this.userId);
    formData.append("desc", this.description);
    formData.append("dataNo", this.number);


    let responseData = await this.$sendDataFile(formData);
    this.imageList.push(responseData.materialImgInfo);
    
},
```
서버에서는 총 4개의 데이터를 받는데 순서대로 파일, 유저 아이디, 설명, 번호 순이다. 파일을 제외한 나머지는 일반 json처럼 데이터를 넣어주듯이 넣어주면 된다.

### axios를 이용해 데이터 전달.

다음은 axios를 이용하여 데이터를 전달해주면 된다. 여기는 크게 기존과는 다르지 않고, 아래 axiosConfig 부분만 주의하면 된다.
```javascript
async function sendDataFile(param) {
    let axiosConfig = {
        headers: {
            "Content-Type": "multipart/form-data",
        }
    }
    let response = await auth.post(commonEnum.BASE_URL + '/regUser',param, axiosConfig);

    return response.data;
}
```
보통 json으로 데이터를 보낼때 Content-Type을 'application/json' 으로 세팅하여 서버쪽을 호출하지만, 파일을 보낼 경우는 컨텐츠 타입을
**multipart/form-data** 로 세팅해주어야 한다. 그렇지 않으면 서버측에서 400에러를 뱉는 것을 볼 수 있을 것이다...


이렇게하면 문제 없이 파일이 업로드 되는 것을 볼 수 있다. formData를 이용하여 저렇게 보낼 데이터를 추가하는 것도 처음해보았고,
파일 업로드 해보는 작업도 처음 해보는거라 생각보다 삽질이 많았다... 나와 같은 고민을 하시는 분들에게 조금이나마 도움이 되었으면 한다.