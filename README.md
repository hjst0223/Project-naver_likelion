# News Translation Service
![home](https://github.com/hjst0223/naver_likelion/blob/main/images/home.PNG)

* 211025-211104

## 1. News Translation Service 소개
Naver의 뉴스 검색 API와 Naver Cloud Platform의 Papago Translation API를 이용해서 사용자가 입력한 키워드에 해당하는 네이버 뉴스를 수집한 후 수집한 뉴스의 원문과 외국어 번역문을 동시에 보여주는 웹서비스로, 특정한 주제의 뉴스를 모아 외국어 번역문을 함께 볼 때 유용하다.


## 2. Server
* Naver Cloud Platform에서 일정 기간 동안 제공 받은 서버 이용
* Flask를 통한 Web Server 실행

## 3. Data
### 3-1. 사용자로부터 키워드 입력받기
```
        <div class="album py-5 bg-light">
            <div class="container">
                <form method=" GET" action="/keyword">
                    <div class="input-group mb-3" style="width:500px;height:50px;margin:auto;">
                        <!-- 뉴스 키워드 입력받기 -->
                        <input type="text" class="form-control" placeholder="원하는 뉴스 키워드를 입력하세요."
                            aria-label="Recipient's username" aria-describedby="button-addon2" , name="keyword">
                        <button class="btn btn-outline-secondary" type="submit" id="button-addon2">검색</button>
                    </div>
                </form>
                <div id="news" class="row row-cols-1 row-cols-sm-2 row-cols-md-1 g-1">
                </div>
            </div>
        </div>
```



### 3-2. Naver의 뉴스 검색 API를 사용하여 뉴스 수집
##### 네이버 뉴스만 번역하도록 key를 추가하여 구분
```
def get_naver_news(display_num, start_num, keywords, client_id, client_secret):
    """
    네이버 뉴스 검색 API 사용해 특정 키워드들의 네이버 뉴스 검색
    :params int display_num: 보여줄 뉴스의 개수
    :params int start_num: 뉴스 시작 번호
    :params list keywords: 키워드 리스트
    :params str client_id: 인증정보
    :params str client_secret: 인증정보
    :return news_items : API 검색 결과 중 뉴스 item들
    :rtype list
    """
    news_items = []
    result = []
    for keyword in keywords:
        # API Request
        # 설정값 세팅하기
        url = 'https://openapi.naver.com/v1/search/news.json'

        sort = 'sim'  # sim: similarity 유사도, date: 날짜
        # display_num = 10

        params = {'display': display_num, 'start': start_num,
                  'query': keyword.encode('utf-8'), 'sort': sort}
        headers = {'X-Naver-Client-Id': client_id,
                   'X-Naver-Client-Secret': client_secret, }

        r = requests.get(url, headers=headers,  params=params)

        # Response 결과
        # 응답결과값(JSON) 가져오기
        # Request(요청)이 성공하면
        if r.status_code == requests.codes.ok:
            result_response = json.loads(r.content.decode('utf-8'))

            result = result_response['items']

            # 네이버 뉴스면 사용하고, 아니면 사용하지 않도록 키 설정하기
            for item in result:
                if 'news.naver.com' in item['link']:
                    item['is_valid'] = 'Y'
                else:
                    item['is_valid'] = 'N'

        # Request(요청)이 성공하지 않으면
        else:
            print('request 실패!')
            failed_msg = json.loads(r.content.decode('utf-8'))
            print(failed_msg)

        news_items.extend(result)

    return news_items
```

### 3-3. 수집한 뉴스의 내용을 web scraping으로 저장
```
def scrape_content(url):
    """
    입력으로 받은 url의 내용을 web scraping
    :params str url: web scraping하고자 하는 url
    :return content : web scraping한 결과
    :rtype str
    """
    content = ''

    # Request 설정값(HTTP Msg) - Desktop Chrome 인 것처럼
    headers = {
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64)AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.86 Safari/537.36'}
    data = requests.get(url, headers=headers)

    soup = BeautifulSoup(data.text, 'html.parser')

    if 'sports.news.naver.com' in url:
        raw_news = soup.select_one('#newsEndContents')
        if not raw_news:
            return content

        for tag in raw_news(['div', 'span', 'p', 'br']):
            tag.decompose()
        content = raw_news.text.strip()

    elif 'news.naver.com' in url:
        raw_news = soup.select_one('#articeBody') or soup.select_one(
            '#articleBodyContents')
        if not raw_news:
            return content

        for tag in raw_news(['div', 'span', 'p', 'br', 'script']):
            tag.decompose()

        content = raw_news.text.strip()

    return content
```

### 3-4. Naver Cloud Platform의 Papago Translation API로 수집한 뉴스를 번역하고 저장
```
def translate(my_content, client_id, client_secret):
    """
    입력으로 받은 my_content를 지정한 외국어로 번역
    :params str my_content: 번역하고자 하는 내용
    :params str client_id: 인증정보
    :params str client_secret: 인증정보
    :return result : 번역한 결과
    :rtype str
    """
    encText = urllib.parse.quote(my_content)

    '''
    번역 가능한 source 언어 - target 언어 (참고 : https://api.ncloud-docs.com/docs/ai-naver-papagonmt-translation)
    : ko(한국어) - en(영어)/ja(일본어)/zh-CN(중국어 간체)/zh-TW(중국어 번체)
    : ko(한국어) - vi(베트남어)/id(인도네시아어)/th(태국어)/de(독일어)
    : ko(한국어) - ru(러시아어)/es(스페인어)/it(이탈리아어)/fr(프랑스어)
    : en(영어) - ko(한국어)/zh-CN(중국어 간체)/zh-TW(중국어 번체)
    : ja(일본어) - ko(한국어)/en(영어)/zh-CN(중국어 간체)/zh-TW(중국어 번체)
    : zh-CN(중국어 간체)/zh-TW(중국어 번체)/vi(베트남어)/id(인도네시아어) - ko(한국어)
    : th(태국어)/de(독일어)/ru(러시아어)/es(스페인어)/it(이탈리아어)/fr(프랑스어) - ko(한국어)
    '''

    data = "source=ko&target=en&text=" + encText
    url = "https://naveropenapi.apigw.ntruss.com/nmt/v1/translation"

    request = urllib.request.Request(url)

    request.add_header("X-NCP-APIGW-API-KEY-ID", client_id)
    request.add_header("X-NCP-APIGW-API-KEY", client_secret)

    response = urllib.request.urlopen(request, data=data.encode("utf-8"))

    rescode = response.getcode()
    if(rescode == 200):
        response_body = response.read()
        result = json.loads(
            response_body.decode('utf-8'))['message']['result']['translatedText']

    else:
        print("Error Code:" + rescode)

    return result
```

```
def save_navernews(keywords):
    """
    입력으로 받은 keywords로 검색한 뉴스 중 네이버 뉴스만 저장
    :params list keywords: 키워드 리스트
    :return naver_news_items :저장한 네이버 뉴스 item들
    :rtype list
    """
    naver_news_items = []
    valid_count = 0
    num = display_num
    start = start_num

    while True:
        news_items = get_naver_news(
            num, start, keywords, client_id, client_secret)

        for item in news_items:
            link = item['link']

            if item['is_valid'] == 'Y':
                content = scrape_content(link)
                item['content'] = content
                item['content'] = content if content != '' else item['description']

            else:
                item['content'] = item['description']

        # item에 번역문 추가하기
        for item in news_items:
            # 네이버 뉴스인 item만 naver_news_items에 저장
            if (item['is_valid'] == 'Y') and (len(item['content']) <= 1000):
                valid_count += 1

                translated_text = translate(
                    item['content'], papago_client_id, papago_client_secret)
                item['translated'] = translated_text
                naver_news_items.append(item)

        if valid_count == display_num:
            break
        else:
            num = display_num - valid_count
            start += display_num

    return naver_news_items
```

## 4. Roadmap
### Backend
* Naver Cloud Platform에서 서버 생성 후 이용
* Naver의 뉴스 검색 API로 뉴스 수집
* Naver Cloud Platform의 Papago Translation API로 수집한 뉴스 번역
* Web Scraping

### Frontend
* 사용자로부터 키워드 입력받기
* Flask를 통한 Web Server 실행

### Release
* Filezila로 서버에 파일 업로드
* Naver Cloud Platform에서 구입한 공인 IP로 서비스 배포


## 5. Demo
### 5-1. 키워드 검색하기


![keyword](https://github.com/hjst0223/naver_likelion/blob/main/images/keyword.gif)




### 5-2. 상위 5개 뉴스와 원문 보기


![link](https://github.com/hjst0223/naver_likelion/blob/main/images/result_link.gif)




### 5-3. 원문과 번역문 보기


![result](https://github.com/hjst0223/naver_likelion/blob/main/images/result.gif)
