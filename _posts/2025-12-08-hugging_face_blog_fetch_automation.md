---
layout: post
title: "n8n을 이용해 블로그 수동적으로 탐험하기"
author: youngjun
categories: [automation]
image: assets/images/blog/posts/2025-12-08-hugging_face_blog_fetch_automation/thumbnail.png
---
* TOC
{:toc}
<!--toc-->
_이 글은 Hugging Face KREW의 자체제작 컨텐츠 입니다._

---
# 들어가며
Hugging Face Krew의 블로그 팀에서 매주 양질의 블로그 글을 번역하고 업로드하고 있지만, 매주 쏟아져나오는 Hugging Face Blog의 모든 글을 번역하기에는 어려움이 있습니다. 그렇다고 매번 언제 글들이 나오는지 찾아 읽어보는것들도 굉장히 힘든 작업입니다. 어떻게 해야 능동적으로 찾아보지 않아도, 블로그에 올라오는 양질의 글을 손대지 않고도 떠먹어볼 수 있을까요?
저는 그 해답을 `자동화`에서 찾았습니다. 매번 블로그에 찾아가고, 새 글이 있는지 확인하고, 블로그를 번역해서 요점만 확인하는 작업을 전부 자동화하는 것이죠. 그래서 이 작업을 n8n을 통해 구현해보고자 합니다.
# n8n이란?
n8n은 2019년 출시된 워크플로우 자동화 플랫폼으로, fair-code 라이센스(Sustainable Use License 및 n8n Enterprise License)로 배포되며, 기술 팀에게 코드의 유연성과 노코드의 편의성을 동시에 제공합니다.
## 주요 특징
- **400개 이상의 통합**: 다양한 앱과 서비스를 연결할 수 있는 사전 구축된 노드를 제공합니다
- **노드 기반 시각적 인터페이스**: 드래그 앤 드롭 방식으로 워크플로우를 구성할 수 있으며, 필요시 JavaScript나 Python 코드를 직접 작성할 수 있습니다
- **AI 네이티브 기능**: LangChain 기반의 AI 에이전트 워크플로우를 구축할 수 있으며, 다양한 LLM, 벡터 스토어, MCP 서버와 통합됩니다
- **배포 옵션**: Self-hosting(온프레미스)과 클라우드 서비스를 모두 제공하여 데이터와 배포에 대한 완전한 제어가 가능합니다
- **활발한 커뮤니티**: GitHub에서 160.6k개의 스타를 받았으며, 100만 이상의 Docker 다운로드와 900개 이상의 워크플로우 템플릿을 보유하고 있습니다
## 배포 옵션 비교
n8n은 무료 커뮤니티 에디션과 유료인 엔터프라이즈 옵션을 모두 제공합니다.
그 중, 로컬 배포가 가능한 커뮤니티 에디션과 클라우드 배포가 되어있는 엔터프라이즈 옵션 중 가장 저렴한 스타터 플랜을 비교해 보겠습니다.

| 구분                      | **Community Edition**                           | **Starter Plan**       |
| ----------------------- | ----------------------------------------------- | ---------------------- |
| **가격**                  | 무료                                              | €20/월 (연간 결제 시)        |
| **호스팅**                 | Self-hosted만 가능                                 | n8n Cloud 호스팅          |
| **워크플로우 실행**            | 무제한                                             | 월 2,500 executions     |
| **활성 워크플로우**            | 무제한                                             | 무제한                    |
| **사용자 수**               | 무제한                                             | 무제한                    |
| **동시 실행**               | 제한 없음                                           | 5개                     |
| **협업 기능**               | ✗ (공유 불가, 생성자만 접근)                              | ○ (1개의 shared project) |
| **워크플로우 히스토리**          | ✗                                               | ○ (1일)                 |
| **AI Workflow Builder** | ✗                                               | ○ (50 credits)         |
| **Insights**            | ✗                                               | ✗                      |
| **Global Variables**    | ✗                                               | ✗                      |
| **지원**                  | Community forum                                 | Forum support          |
| **등록 시 추가 기능***         | Folders, Debug in editor, Custom execution data | -                      |

# n8n 커뮤니티 에디션 셀프 호스팅 해보기
n8n의 엔터프라이즈 옵션을 사용할 때에는 처음 14일간 무료로 제공되지만, 이후에는 월간 결제를 해야 하기 때문에 미니PC를 이용해 커뮤니티 에디션으로 사용해보고자 합니다.
다음은 제가 구동했던 미니PC의 스펙입니다.

| 구분      | 상세 스펙                           |
| ------- | ------------------------------- |
| CPU     | Intel 14th N150 4Core Processor |
| Storage | 512GB NVMe M.2 SSD              |
| RAM     | 16GB DDR4 3200MHz               |
| OS      | Windows 11                      |

## Windows OS에서 Linux-like 환경 실행하기
n8n은 Docker 또는 npm 런타임에서 최적으로 실행할 수 있습니다. Windows 환경에서 Docker Desktop을 실행하기 위해서는 Windows에서 Linux-like 환경을 실행할 수 있게끔 하는 WSL2가 필요합니다.
Powershell에서 다음 명령어로 WSL2를 설치하고, Docker Desktop Application을 다운로드합니다.
```shell
wsl --install
```

이제 명령어로 n8n을 다운받고 실행해 봅시다.
```shell
docker run -d --restart always --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
```
명령어를 설명하자면 다음과 같습니다.
- docker run : 컨테이너 실행
- -d : 컨테이너를 백그라운드로 항상 실행
- --restart always : PC가 켜질때마다 무조건 다시 시작
- --name n8n : 컨테이너의 이름으로 n8n을 사용
- `-p 5678 : 5678 ` : 내 PC의 5678번 포트를 컨테이너의 5678번 포트로 연결
- `-v n8n_data : /home/node/.n8n ` : 내 PC에 `n8n_data`라는 데이터 저장소를 만들고, n8n 컨테이너의 데이터 폴더와 연결(영구적 데이터 생성)
- `docker.n8n.io/n8nio/n8n`: 이 주소에 있는 n8n 이미지를 사용해서 실행

실행이 되었으면, `localhost:5678`로 컨테이너에서 잘 실행되고 있는지 확인해 봅시다.
## Docker-Compose
실행이 잘 되었다면, 이제 조금 더 명령어 기반이 아닌, config 파일 기반으로 복잡한 명령어를 설정하더라도 더 편리한 실행 환경을 만들기 위해 config 파일을 만든 Docker-Compose로 서버를 간략하게 실행해보겠습니다.

먼저 리눅스 홈 디렉토리에 접근하여 가장 접근성 좋게 디렉토리를 만들어봅시다.
```shell
cd ~
mkdir n8n-docker
cd n8n-docker
```

디렉토리를 만들었다면, docker-compose.yaml 파일을 만들어 봅시다.
```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n  # 사용하시던 이미지 그대로 사용
    container_name: n8n             # --name n8n 대응
    restart: always                 # --restart always 대응
    ports:
      - "5678:5678"                 # -p 5678:5678 대응
    environment:
      - N8N_HOST=localhost
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - NODE_ENV=production
      - GENERIC_TIMEZONE=Asia/Seoul # (추천) 한국 시간대 설정 추가
      - TZ=Asia/Seoul               # (추천) 시스템 시간대 설정 추가
    volumes:
      - n8n_data:/home/node/.n8n    # 기존 데이터 연결

volumes:
  n8n_data:
    external: true  # "새로 만들지 말고, 이미 있는 'n8n_data'를 써라"라는 뜻입니다.
```

파일을 만들었으면, 해당 파일이 있는 위치로 들어가서 아래 명령어 한줄이면 위의 명령어를 전부 포함해서 docker 컨테이너가 올라가게 됩니다.
```bash
docker compose -d
```
## ngrok로 터널링하기 - 추가
물론 localhost에서는 성공적으로 실행했지만, 미니 PC가 아닌 외부에서도 내 n8n에 접근할 수 있게 하기 위해서는 도메인이 필요합니다. 외부 서비스에서 n8n으로 데이터를 쏠 필요가 있을 경우에도 필수적으로, 다음과 같은 경우에서 사용됩니다.
- **Webhook 사용**: 외부 서비스(GitHub, Slack 등)에서 n8n으로 데이터를 전송해야 할 때
- **모바일 접근**: 스마트폰에서 워크플로우를 확인하거나 실행할 때
- **원격 작업**: 집이 아닌 다른 장소에서 n8n에 접근할 때
이런 터널링 프로그램에는 몇 가지가 있습니다.
- Cloudflare
- Tailscale
- ngrok
이번 포스트에서는 ngrok을 이용하여 내 홈 서버의 n8n을 외부에 노출시켜보려고 합니다.
ngrok 또한 cli에서 설치 및 실행할 수 있습니다.
조금 전에 이용했던 Docker-compose를 이용하여 실행해 봅시다.

```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    container_name: n8n
    restart: always
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=n8n
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - NODE_ENV=production
      - GENERIC_TIMEZONE=Asia/Seoul
      - TZ=Asia/Seoul
      # 중요: 나중에 ngrok URL이 나오면 n8n에게 알려줘야 완벽해집니다.
      # 지금은 비워두거나 무시해도 됩니다.
    volumes:
      - n8n_data:/home/node/.n8n

  ngrok:
    image: ngrok/ngrok:latest
    container_name: ngrok
    restart: unless-stopped
    command:
      - "http"
      - "n8n:5678"  # "n8n 컨테이너의 5678 포트로 터널을 뚫어라"
    ports:
      - "4040:4040" # ngrok 상태 확인 페이지
    environment:
      - NGROK_AUTHTOKEN=여기에_아까_복사한_토큰을_붙여넣으세요

volumes:
  n8n_data:
    external: true
```

그리고 다시 터미널에서 변경 사항을 적용합니다.
```bash
docker compose up -d
```
그러면 `Running 2/2` (n8n, ngrok) 메시지가 보일 것입니다.

이제, `localhost:4040`(ngrok 상태창)에 접속해서 Endpoints를 확인하면 다음의 화면이 보이고, 여기서 보이는 Endpoint를 이용해 외부에서 들어가 봅시다.(보통 핸드폰의 셀룰러 데이터 환경에서 들어가보는게 가장 확실합니다!)
![](https://i.imgur.com/KkYMT3o.png)

# RSS Feed
n8n을 성공적으로 구동하였다면, 다음으로는 본격적으로 자동화를 구현해 볼 시간입니다. 블로그를 자동으로 fetch하려는 경우 몇 가지 방법이 있지만, RSS 방식을 이용해 보겠습니다. RSS(Really Simple Syndication 또는 Rich Site Summary)는 웹사이트의 최신 콘텐츠를 구독자에게 자동으로 전달하는 표준 형식입니다. 블로그, 뉴스 사이트, 팟캐스트 등이 새로운 게시물을 발행하면, RSS 피드를 통해 구독자들이 자동으로 업데이트를 받을 수 있습니다. 
## RSS의 작동 원리
RSS는 XML 형식으로 구성된 파일로, 다음과 같은 정보를 포함합니다:
```xml
<?xml version="1.0" encoding="UTF-8" ?> 
<rss version="2.0"> 
    <channel> 
        <title>블로그 제목</title> 
        <link>https://example.com</link> 
        <description>블로그 설명</description> 
        <item> 
            <title>최신 게시글 제목</title>
            <link>https://example.com/post/1</link> 
            <description>게시글 요약...</description> 
            <pubDate>Mon, 08 Dec 2025 10:00:00 GMT</pubDate> 
        </item> 
    </channel> 
</rss>
```

## RSS의 장점 
**1. 중앙 집중식 정보 수집** 
- 여러 웹사이트를 일일이 방문하지 않고 한 곳에서 모든 업데이트 확인 
- Feedly, Inoreader 같은 RSS 리더로 편리하게 관리 
**2. 알고리즘 없는 순수한 콘텐츠** 
- 소셜 미디어와 달리 알고리즘에 의해 필터링되지 않음 
- 구독한 모든 콘텐츠를 시간순으로 받아볼 수 있음 
**3. 자동화에 최적**
- 표준화된 형식으로 프로그래밍적 처리가 쉬움 
- n8n 같은 자동화 도구와 완벽하게 호환 
그러면 우리가 받고싶은 HF 블로그의 실제 [RSS 피드](https://huggingface.co/blog/feed.xml)는 어떻게 생겼는지 확인해 볼까요?

# n8n을 이용해서 HF Blog 떠먹기, 그런데 이제 hugging-face model을 곁들인...
이제 본격적으로 n8n을 이용해 워크플로우를 구성해 보겠습니다. n8n의 워크플로우는 어떻게 구성해야 할까요?![](https://i.imgur.com/aXZd0ra.png)

## 워크플로우 구조
### n8n의 기본 구성 요소: 노드와 엣지
n8n 워크플로우는 **노드(Node)** 와 **엣지(Edge)** 로 구성됩니다.

**노드(Node)**
- 워크플로우에서 **특정 작업을 수행하는 단위**입니다.
- 각 노드는 데이터를 입력받아 처리한 후 다음 노드로 전달합니다.
- 예시:
    - **RSS Feed Trigger**: RSS 피드를 읽어오는 노드
    - **HTTP Request**: 웹 페이지를 가져오거나 API를 호출하는 노드
    - **Code**: JavaScript/Python 코드를 실행하는 노드
    - **Notion**: Notion API로 데이터를 저장하는 노드

**엣지(Edge)**
- 노드와 노드를 **연결하는 선**입니다.
- 데이터가 흐르는 경로를 정의합니다.
- 한 노드의 출력(Output)이 다음 노드의 입력(Input)으로 전달됩니다.
각 노드는 JSON 형식의 데이터를 주고받으며, 이전 노드의 결과를 `$json`, `$input` 등으로 참조할 수 있습니다.

## 워크플로우 설계
가장 처음 해야 할 작업은, 내 워크플로우를 어떻게 구성할 것인지를 생각하고 구현해내야 합니다.
우리의 목표는 **'Hugging Face Blog의 RSS Feed를 받아와서, 새 글이 있다면 내용을 파싱한 뒤, 요약 및 번역하여 Notion 지식베이스에 업로드하기'** 이므로, 이를 단계별로 나누면 다음과 같습니다:

1. **RSS 피드 읽기**: HF 블로그의 RSS 피드를 확인하고, 새 글이 있다면 아래 작업을 실시한다.
2. **콘텐츠 추출**: 글의 링크로 들어가서 본문 내용을 정제한다.
3. **AI 처리**: 정제된 내용을 HuggingFace 모델로 요약하고 한국어로 번역한다.
4. **저장**: 만들어진 컨텐츠를 Notion 데이터베이스에 저장한다.

이제 각 단계를 n8n 노드로 구현해보겠습니다.
### RSS 피드 읽기
먼저, RSS Feed trigger로 RSS 피드를 추적하고 있다가, 새 글이 올라오면 정보를 받아와야 합니다.
```yaml
Type: RSS Feed trigger
Parameters: # 이 부분은 본인에 맞게 커스터마이징 해보세요!
    Mode: Every Day
    Hour: 14
    Minute: 0
Feed URL: https://huggingface.co/blog/feed.xml
```
파라미터 작성 후 `Fetch Test Event`를 눌러보면 아래의 화면이 성공적으로 나와야 합니다!
![](https://i.imgur.com/u9UhNqj.png)

### 컨텐츠 받아오고, 추출하기
RSS Feed의 xml 파일에 description 태그로 요약본이 제공되는 곳도 있지만, 보통은 제공되지 않습니다. 따라서 해당 내용을 확인하기 위해서는 link 태그로 감싸져 있는 링크에서 HTML을 받아온 뒤, 본문을 추출(파싱)하는 작업이 꼭 필요합니다.

**HTML 받아오기**
```yaml
Type: HTTP Request
Parameters:
    Method: GET
    URL: {{$json.link}} # n8n은 이전의 output을 토대로, link를 가져올 수 있습니다!
    Authentication: None
    Send Headers: On
        Specify Headers: Using Fields Below
        Header Parameters:
            Name: Accept
            Value: text/html
```

**파싱**
```yaml
Type: Code in JavaScript
Parameters:
    Mode: Run Once for All Items
    Language: Javascript
    JavaScript: # 여기에 아래의 예제를 참고하여 본인만의 파싱 로직을 작성하세요!
```

아래는 자바스크립트 코드의 예제입니다!
```javascript
const html = $input.first().json.data;

if (!html || html.length === 0) {
  return { error: "HTML 없음", htmlLength: 0 };
}
const divStart = html.lastIndexOf('<div', blogContentIndex);
  if (divStart === -1) {
    return { error: "blog-content div 태그를 찾을 수 없음" };
  }

  let content = '';
  let divCount = 0;
  let inContent = false;
  let i = divStart;

  while (i < html.length) {
    if (html.substring(i, i + 4) === '<div') {
      divCount++;
      inContent = true;
      i = html.indexOf('>', i) + 1;
      continue;
    }
    
    if (html.substring(i, i + 6) === '</div>') {
      divCount--;
      if (inContent && divCount === 0) {
        content = html.substring(html.indexOf('>', divStart) + 1, i);
        break;
      }
      i += 6;
      continue;
    }

    i++;
  }

  if (!content) {
    return { error: "blog-content 콘텐츠를 추출할 수 없음" };
  }

  // 2. 불필요한 div 제거
  
  // 2.1 SVELTE 마커 제거
  content = content.replace(/<div[^>]*SVELTE_HYDRATER[^>]*>[\s\S]*?<\/div>/gi, '');
  
  // 2.2 주석 제거
  content = content.replace(/<!--[\s\S]*?-->/g, '');
  
  // 2.3 SVG 제거
  content = content.replace(/<svg[^>]*>[\s\S]*?<\/svg>/gi, '');
  
  // 2.4 스크립트, 스타일 제거
  content = content.replace(/<script[^>]*>[\s\S]*?<\/script>/gi, '');
  content = content.replace(/<style[^>]*>[\s\S]*?<\/style>/gi, '');
  
  // 2.5 특정 클래스 div 제거
  const classesToRemove = [
    'SVELTE_HYDRATER',
    'not-prose',
    'upvote',
    'author',
    'byline',
    'sticky',
    'absolute',
    'flex-none',
    'peer-focus-within',
    'invisible',
    'RepoCodeCopy',
    'UpvoteControl',
    'BlogAuthorsByline',
    'DeviceProvider',
    'SystemThemeMonitor',
    'flex items-center font-sans',
    'mb-12 flex',
    'header-link'
  ];
  
  classesToRemove.forEach(cls => {
    const escapedCls = cls.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
    const regex = new RegExp(`<div[^>]*class="[^"]*${escapedCls}[^"]*"[^>]*>[\\s\\S]*?<\\/div>`, 'gi');
    content = content.replace(regex, '');
  });
  
  // 2.6 다른 불필요한 요소
  content = content.replace(/<button[^>]*>[\s\S]*?<\/button>/gi, '');
  content = content.replace(/<nav[^>]*>[\s\S]*?<\/nav>/gi, '');
  content = content.replace(/<iframe[^>]*>[\s\S]*?<\/iframe>/gi, '');
  content = content.replace(/<input[^>]*>/gi, '');
  
  // 2.7 이미지 텍스트만 추출
  content = content.replace(/<img[^>]*alt="([^"]*)"[^>]*>/gi, '$1');
  content = content.replace(/<img[^>]*>/gi, '');

  // 3. 코드 블록 보존
  const codeBlocks = [];
  
  // 3.1 <pre><code> 형식
  content = content.replace(/<pre[^>]*>[\s\S]*?<code[^>]*>([\s\S]*?)<\/code>[\s\S]*?<\/pre>/gi, 
    (match, code) => {
      const decoded = code
        .replace(/&lt;/g, '<')
        .replace(/&gt;/g, '>')
        .replace(/&amp;/g, '&')
        .replace(/&quot;/g, '"')
        .replace(/&#x27;/g, "'")
        .replace(/<span[^>]*>/g, '')
        .replace(/<\/span>/g, '')
        .trim();
      
      if (decoded && decoded.length > 10) {
        codeBlocks.push(`\n\`\`\`\n${decoded}\n\`\`\`\n`);
        return `__CODE_BLOCK_${codeBlocks.length - 1}__`;
      }
      return '';
    }
  );

  // 3.2 <code> 단독
  content = content.replace(/<code[^>]*>([\s\S]*?)<\/code>/gi, (match, code) => {
    const decoded = code
      .replace(/&lt;/g, '<')
      .replace(/&gt;/g, '>')
      .replace(/&amp;/g, '&')
      .replace(/&quot;/g, '"')
      .replace(/<span[^>]*>/g, '')
      .replace(/<\/span>/g, '')
      .trim();
    if (decoded && decoded.length > 5) {
      return ` \`${decoded}\` `;
    }
    return '';
  });

  // 4. 모든 HTML 태그 제거
  content = content.replace(/<[^>]+>/g, ' ');

  // 5. 코드 블록 복원
  codeBlocks.forEach((block, index) => {
    content = content.replace(`__CODE_BLOCK_${index}__`, block);
  });

  // 6. HTML 엔티티 디코딩
  content = content
    .replace(/&nbsp;/g, ' ')
    .replace(/&amp;/g, '&')
    .replace(/&lt;/g, '<')
    .replace(/&gt;/g, '>')
    .replace(/&quot;/g, '"')
    .replace(/&#39;/g, "'")
    .replace(/&#x27;/g, "'")
    .replace(/&apos;/g, "'");

  // 7. 마크다운 이미지 태그 제거
  content = content.replace(/!\[.*?\]\(.*?\)/g, '');

  // 8. 텍스트 정리
  content = content
    .replace(/\s+/g, ' ')
    .replace(/\n\s*\n/g, '\n\n')
    .replace(/^ +| +$/gm, '')
    .trim();

  // 9. 검증
  if (!content || content.length < 200) {
    return {
      error: "파싱된 콘텐츠가 너무 짧음",
      debug: {
        length: content.length,
        preview: content.substring(0, 300)
      }
    };
  }
  
  return {
    success: true,
    parsedContent: content,
    contentLength: content.length,
    codeBlockCount: codeBlocks.length,
    originalLength: html.length,
  };

} catch (error) {
  return {
    error: `파싱 오류: ${error.message}`,
    errorType: error.name
  };
}
```

### 요약하기
이제, Hugging Face의 Model을 이용해서 파싱한 내용을 요약해 보겠습니다.
Hugging Face와 관련된 노드는 `Hugging Face Inference Model`  노드도 있지만, 이는 채팅 전용 모델로, 이번에는 API request 방식으로 모델 호출을 시도해 보겠습니다.

**Authentication**
요청을 보내기 위해서는 인증된 토큰을 보내야 하는데, Hugging Face 모델을 사용하기 위해서는 Hugging Face API 토큰이 필요합니다. 해당 토큰은 아래처럼 Hugging Face 홈페이지에서 Account 페이지의 Access Tokens 탭에서 발급받을 수 있습니다. 
![](https://i.imgur.com/rCIzKYV.png)
여기서 발급받은 토큰은 노드의 `Predefined Credential Type`에서 `HuggingFaceApi account`인증을 통해 지속적으로 사용할 수 있습니다.
![](https://i.imgur.com/ogytusC.png)

아래는 요약 노드를 정리했습니다. 저는 실험을 위해  `Meta-Llama-3.3-70B-Instruct` 모델을 사용했지만, 실험해보고 싶은 여러 모델을 사용해 보세요!
```yaml
Type: HTTP Request
Parameters:
    Method: POST
    URL: https://router.huggingface.co/sambanova/v1/chat/completions
    Authentication: Predefined Credential Type
    Credential Type: HuggingFaceApi
    HuggingFaceApi: HuggingFaceApi account
    Send Headers: On
        Specify Headers: Using Fields Below
        Header Parameters:
            Name: Content-Type
            Value: application/json
    Send Body: On
        Body Content Type: json
        Specify Body: Using JSON
        JSON: 
            \{\{JSON.stringify{ # '\'를 지워 주세요.
                model: "Meta-Llama-3.3-70B-Instruct",
                messages: [{
                    role: "user",
                    content: "다음 영문 기술 블로그 글을 한국어로 5-7문장으로 핵심만 요약해줘. 기술 용어는 그대로 유지해:\n\n" + \{\{$json.content}} # '\'를 지워 주세요.
                }],
                max_tokens: 800,
                temperature: 0.3}
            }}

```

노드를 실행해 output이 잘 출력되는지 확인합니다.
![](https://i.imgur.com/QyjNao7.png)

### 저장하기
이제 마지막으로, 이 content를 지식베이스에 저장해 봅시다. 대표적인 데이터베이스를 이용할 수 있는 `Notion`을 이용해서 저장해 보겠습니다.
노션에서 페이지를 하나 만들고, 데이터베이스를 하나 생성한 뒤, 데이터베이스 페이지에 접속합니다.( 데이터베이스에서`데이터베이스 보기` 메뉴를 통해 접근할 수 있습니다)
[API 통합](https://www.notion.so/profile/integrations) 페이지에서 API 키를 발급받은 다음(주의 : 꼭 I/O 기능을 추가해야 합니다!) 저장하고, 데이터베이스에 연결해주면 절반의 성공입니다.
![](https://i.imgur.com/ypZa0Td.png)
아래 그림처럼 연결되었다면, 성공한 겁니다!
![](https://i.imgur.com/HzC3qvt.png)

이제 노션 데이터베이스에 반영하기 위해, 필드들을 수정합니다.

| 속성 이름   | 타입           | 용도       |
| ------- | ------------ | -------- |
| Title   | Title        | 블로그 글 제목 |
| Summary | Text         | 한국어 요약   |
| URL     | URL          | 원본 링크    |
| Date    | Date         | 발행일      |

마지막으로, 다시 n8n 페이지에 돌아가서 노드를 수정합니다.

```yaml
Type: Notion
Parameters:
    Credential to connect with: Notion account
    Resource: Database Page
    Operation: Create
    Database: #여기에 드롭다운으로 본인의 노션 데이터베이스를 선택합니다.
    Title: \{\{ $('RSS Feed Trigger').item.json.title }} # '\'를 지워 주세요.
    Simplify: On
    Properties:
        Key Name or ID: summary
            Rich Text: On
            text:
                Type: text
                Text: \{\{ $json.choices[0].message.content }} # '\'를 지워 주세요.
        Key Name or ID: url
            URL: \{\{ $('RSS Feed Read').item.json.link }} # '\'를 지워 주세요.
        Key Name or ID: Published Date
            Include Time: On
            Date: \{\{ $('RSS Feed Read').item.json.isoDate }} # '\'를 지워 주세요.
```

아래 그림처럼 성공적으로 내용이 저장되었다면, 이제 정말 완료입니다!
![](https://i.imgur.com/IUDf4M5.png)

# 마무리
n8n은 커뮤니티 버전이라도 정말 많은 노드와 기능들을 포함하고 있습니다. 여기에 없는 것은 당신의 상상력 뿐입니다. 예시로 보여드린것 외에도, 충분히 더 좋은 작업을 구현해볼 수 있을 것이라고 생각합니다. 저는 아래처럼, 요약 뿐 아니라 추출한 내용을 기반으로 내용을 잘 이해했는지 체크하기 위해 문제를 내고, 문제-답변 질의 세트를 추가했습니다.
![](https://i.imgur.com/c4WvxZC.png)
물론 엔터프라이즈 버전에 비해 몇몇 제약이 있을수도 있지만, n8n 커뮤니티 버전을 이용하여 여러분들의 귀찮은 작업들을 자동화하고, 더 나아가 상상력을 구현할 수 있는 기회가 되었으면 좋겠습니다!