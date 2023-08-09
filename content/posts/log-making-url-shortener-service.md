---
title: "FastAPI로 단축 도메인 서비스 만들기"
date: 2023-08-09T10:10:11+09:00
draft: true
tags: [FastAPI, PostgreSQL, React, 단축 도메인, URL 단축, URL shortener]
---

단축 도메인 서비스를 만들었습니다.   
단축 도메인 서비스는 긴 URL을 짧은 URL에 연결시켜주는 서비스입니다.   
생각보다 로직이 간단해서 단축 도메인 기능 만들기는 쉬웠지만 제가 코린이인지라 개발 환경 세팅과 데이터베이스 연결, 배포 등 많은 시행착오가 있었습니다.   

# 기술 스택
- Python, FastAPI
- PostgreSQL
- React

# 백엔드
백엔드는 FastAPI를 이용했습니다.   
Flask만큼 쉽고 간단한데다 더 빠르기 때문입니다.   
API 문서 자동으로 만들어주는 기능은 기가 막합니다.   
문서에서 API 테스트도 할 수 있습니다.

데이터베이스는 PostgreSQL을 이용했습니다.   
배포는 도커를 이용했습니다.

백엔드를 만들면서 가장 힘들었던 부분은 개발환경 세팅과 배포가 아니었을까 합니다.   
이걸 만들면서 도커와 도커 컴포즈를 처음 접했습니다...!   
데이터베이스도 굉장히 어려웠습니다.   
개발환경 세팅하고 데이터베이스 연결하는데 대부분의 시간을 보낸 것 같습니다.

# 백엔드 메인 로직
메인 로직은 간단합니다.   
1. 데이터베이스에 랜덤한 문자열 키와 원본 URL을 저장한다.
2. 문자열 키는 `https://example.tld/<문자열 키>` 와 같이 단축된 링크에 사용된다.
3. 단축된 링크를 열면 문자열 키에 연결된 원본 URL을 데이터베이스에서 꺼내 리디렉트 시킨다.

아래는 위의 로직이 있는 소스 코드의 일부입니다.   
주석으로 설명을 넣었습니다.
FastAPI를 이용했습니다.

```python
# URL을 단축시켜 데이터베이스에 저장
@app.post("/api/urls", response_model=schemas.ShortenedURL)
def create(url: schemas.URLToShorten, db: Session = Depends(get_db)):
    # 올바른 URL이 아니면 400 오류 반환
    if not validators.url(url.url):
        raise HTTPException(status_code=400, detail="Invalid URL")
    
    # 데이터베이스에 있는 URL과 키 가져오기
    shortened_url = crud.get_shortened_url(db, url.url)
    
    if shortened_url is None: # 데이터베이스에 없다면 키를 새로 생성하고 데이터베이스에 저장
        string = "abcdefghijklmnopqrstuvwxyz"
        string += string.upper()
        string += "0123456789"
        string = string * KEY_LEN

        key = "".join(random.sample(string, KEY_LEN))

        # 데이터베이스에 중복된 키가 있는지 확인
        while not crud.get_original_url(db, key) is None:
            key = "".join(random.sample(string, KEY_LEN))

        shortened_url = crud.shorten_url(db, url.url, key)

        return {
            "key": key,
            "original_url": url.url
        }
    
    return shortened_url


# 단축된 URL을 열었을 때 원래 URL로 리디렉트
@app.get("/{key}")
def redirect(key: str, db: Session = Depends(get_db)):
    # 데이터베이스에서 키에 맞는 URL 꺼내기
    shortened_url = crud.get_original_url(db, key)

    if shortened_url is None: # 데이터베이스에 없다면 404 오류 반환
        raise HTTPException(status_code=404, detail="URL is not found.")
    
    # 원래 URL을 301 리디렉트
    return RedirectResponse(shortened_url.original_url, status_code=301)
```

이 코드는 일부분이고 자세한 내용은 깃허브 저장소를 확인할 수 있습니다.   
https://github.com/sunwoo1524/krll-backend

# 백엔드 배포
백엔드와 프론트엔드를 분리 개발했습니다.   
그래서 배포 또한 백엔드는 GCP 프리 티어에서, 프론트엔드는 Vercel에서 했습니다.   
프리 티어 VPS로 오라클 클라우드도 있는데 가입이 죽어도 안돼 포기했습니다.   
가입이 안돼 문의 이메일을 남겼더니 꺼지라고 했다는 사람도 있는 거 보면 가입할 생각이 싹 사라집니다.   
결국 GCP에도 평생 무료 서버가 있다는 것을 듣고 GCP를 선택했는데 얘는 자동 결제가 멋대로 될 것 같아 불안합니다... ~~결국 답은 돈~~   

어떤 분이 클라우드플레어 CDN을 이용하면 https 인증서까지 무료로 이용할 수 있다고 하여 해봤습니다.  
사실 인증서 발급과 적용도 배포만큼 어려웠습니다...

# 프론트엔드
프론트엔드는 리액트를 사용했습니다.   
그냥... 리액트가 가장 무난하고 많이 쓰여서? 선택했습니다.   
API로 단축된 URL을 받고 QR 코드로 만들어주는 기능을 넣었습니다.   
이용 규칙도 명시했습니다.   
배포는 Vercel을 이용했습니다.   
프론트엔드 깃허브 저장소도 공개되어 있습니다.   
https://github.com/sunwoo1524/krll-frontend

# 끝으로
이 프로젝트를 진행하면서, 정말 많은 것을 배운 것 같습니다.   
도커와 도커 컴포즈 사용법, SQL 문법, 백엔드와 데이터베이스의 연결, 백엔드와 프론트엔드의 분리, 리눅스 서버에서 백엔드 배포 그리고 왜 윈도우가 쓰레기같은지 등...   
방학이 끝나기 전 성공적으로 프로젝트를 완성하고 배포해서 기분이 정말 좋습니다.   
깃허브 저장소에 이 프로젝트의 모든 코드가 공개되어 있습니다.   
혹시 제 부족한 소스코드가 궁금하신 분은 언제든지 구경하고 참고하고 복붙하셔도 됩니다.

끝으로 제가 만든 서비스를 홍보하겠습니다.   
긴 글 읽어주셔서 정말 감사드리고 댓글은 언제든지 환영입니다.   
다음 글에서 뵙겠습니다!

...

## Krll, 개인정보 수집 없는 간편한 단축 도메인 서비스
로그인과 회원가입이 필요 없습니다!   
광고가 나오는 중간 페이지가 없습니다!   
개인정보 수집 없는 빠르고 편리한 단축 도메인 서비스를 만나보세요!   
로그인이 필요 없으니 당연히 무료입니다!   
링크 단축과 함께 QR 코드가 생성되어 스마트폰으로도 편하게 열 수 있습니다!   
개인정보 수집 없는 간편한 단축 도메인 서비스 Krll을 지금 만나보세요!   
https://www.krll.me