# BookBite GitHub + Vercel 구현 전략

## 개요

이 문서는 도서 판매 중개 플랫폼 BookBite를 GitHub와 Vercel을 활용하여 구현하는 전략을 설명합니다. 기존 WordPress 기반 사이트에서 모던 웹 개발 환경으로 전환하여 안정성, 성능, 유지보수성을 향상시키는 방안을 제시합니다.

## 목차

1. [프로젝트 분석](#1-프로젝트-분석)
2. [기술 스택 선정](#2-기술-스택-선정)
3. [GitHub + Vercel 아키텍처](#3-github--vercel-아키텍처)
4. [데이터 수집 및 관리](#4-데이터-수집-및-관리)
5. [프론트엔드 구현](#5-프론트엔드-구현)
6. [관리자 인터페이스](#6-관리자-인터페이스)
7. [배포 및 운영 전략](#7-배포-및-운영-전략)
8. [마이그레이션 로드맵](#8-마이그레이션-로드맵)
9. [비용 분석](#9-비용-분석)
10. [결론 및 기대효과](#10-결론-및-기대효과)

## 1. 프로젝트 분석

### BookBite 사이트 목표
- **베스트셀러 통합 플랫폼**: 한국 주요 인터넷 서점(예스24, 알라딘, 교보문고)의 베스트셀러 정보를 한 곳에서 제공
- **도서 정보 중개**: 사용자에게 도서 정보를 제공하고 해당 서점으로 연결
- **수익 모델**: 제휴 마케팅을 통한 커미션 수익 창출
- **데이터 수집**: 각 서점의 베스트셀러, 도서 리뷰, 관련 유튜브 콘텐츠, 도서 뉴스 수집

### 주요 기능 요구사항
1. 베스트셀러 순위 정보 통합 제공
2. 도서 상세 정보 제공 (표지, 저자, 출판사, 가격 등)
3. 서점별 바로가기 링크 제공
4. 주기적인 데이터 수집 및 업데이트
5. 관리자용 데이터 수집 인터페이스
6. 사용자 친화적 UI/UX 제공

### 현 시스템 문제점
- PHP 기반 WordPress 사이트의 안정성 문제
- 데이터 수집 및 처리 과정의 오류 발생
- 중복 코드 및 유지보수 어려움
- 성능 및 보안 취약점

## 2. 기술 스택 선정

### 프론트엔드
- **Next.js**: React 기반 프레임워크로 SSR/SSG 지원
- **TypeScript**: 타입 안정성 확보
- **Tailwind CSS**: 유틸리티 기반 반응형 디자인
- **React Query**: 서버 상태 관리 및 데이터 페칭

### 백엔드
- **Next.js API Routes**: 서버리스 함수로 API 구현
- **Prisma/Mongoose**: 데이터베이스 ORM
- **MongoDB Atlas/Vercel KV**: 데이터 저장소

### 데이터 수집
- **Node.js**: 크롤링 및 API 스크립트
- **Cheerio/Puppeteer**: 웹 크롤링
- **YouTube Data API**: 유튜브 영상 데이터
- **News API**: 도서 관련 뉴스

### 인프라
- **GitHub**: 소스코드 관리 및 CI/CD
- **Vercel**: 호스팅 및 배포 자동화

## 3. GitHub + Vercel 아키텍처

### GitHub 저장소 구조

```
bookbite/
├── .github/                 # GitHub Actions 워크플로우
│   └── workflows/           # 자동화 스크립트
├── pages/                   # Next.js 페이지
│   ├── api/                 # 서버리스 API 함수
│   ├── admin/               # 관리자 페이지
│   └── ...                  # 일반 페이지
├── components/              # 재사용 컴포넌트
├── lib/                     # 유틸리티 함수
│   ├── collectors/          # 데이터 수집기
│   └── db/                  # 데이터베이스 연결
├── public/                  # 정적 파일
├── styles/                  # CSS 스타일
├── types/                   # TypeScript 타입 정의
├── scripts/                 # 빌드, 데이터 수집 스크립트
└── next.config.js           # Next.js 설정
```

### Vercel 배포 환경

- **개발 환경**: 로컬 개발 서버
- **스테이징 환경**: 풀 리퀘스트별 자동 배포 (https://pr-[number].bookbite.vercel.app)
- **운영 환경**: 메인 브랜치 배포 (https://bookbite.vercel.app)

### CI/CD 파이프라인

1. GitHub에 코드 푸시
2. GitHub Actions 실행 (린팅, 테스트)
3. Vercel 자동 빌드 및 배포
4. 배포 결과 알림 (Slack/Discord)

## 4. 데이터 수집 및 관리

### 데이터 수집 자동화

#### 방법 1: Vercel Cron Jobs (Pro 플랜)

```typescript
// pages/api/cron/collect-bestsellers.ts
import type { NextApiRequest, NextApiResponse } from 'next'
import { collectYes24, collectAladin, collectKyobo } from '../../../lib/collectors'

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  // Cron 인증 확인
  if (req.headers.authorization !== `Bearer ${process.env.CRON_SECRET}`) {
    return res.status(401).json({ error: 'Unauthorized' })
  }

  try {
    // 각 서점 데이터 수집
    const [yes24Books, aladinBooks, kyoboBooks] = await Promise.all([
      collectYes24(),
      collectAladin(),
      collectKyobo()
    ])
    
    // 데이터 저장 로직...
    
    res.status(200).json({ 
      success: true, 
      count: yes24Books.length + aladinBooks.length + kyoboBooks.length 
    })
  } catch (error) {
    res.status(500).json({ error: 'Collection failed' })
  }
}
```

#### 방법 2: GitHub Actions

```yaml
# .github/workflows/collect-data.yml
name: Collect Book Data

on:
  schedule:
    - cron: '0 0 * * *'  # 매일 자정
  workflow_dispatch:     # 수동 실행 옵션

jobs:
  collect:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          
      - name: Install dependencies
        run: npm ci
        
      - name: Collect bestseller data
        run: node scripts/collect-bestsellers.js
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
          
      - name: Commit and push changes
        run: |
          git config --global user.name 'GitHub Action'
          git config --global user.email 'action@github.com'
          git add data/
          git diff --quiet && git diff --staged --quiet || git commit -m "Update bestseller data"
          git push
```

### 데이터 저장 옵션

#### MongoDB Atlas

```typescript
// lib/mongodb.ts
import { MongoClient } from 'mongodb'

const uri = process.env.MONGODB_URI
const options = {}

let client
let clientPromise

if (!process.env.MONGODB_URI) {
  throw new Error('MongoDB URI not found')
}

if (process.env.NODE_ENV === 'development') {
  if (!global._mongoClientPromise) {
    client = new MongoClient(uri, options)
    global._mongoClientPromise = client.connect()
  }
  clientPromise = global._mongoClientPromise
} else {
  client = new MongoClient(uri, options)
  clientPromise = client.connect()
}

export default clientPromise
```

#### Vercel KV (Redis)

```typescript
// lib/kv.ts
import { kv } from '@vercel/kv'

export async function saveBestsellers(data) {
  await kv.set('bestsellers', JSON.stringify(data))
  await kv.expire('bestsellers', 60 * 60 * 24) // 24시간 유효
}

export async function getBestsellers() {
  const data = await kv.get('bestsellers')
  return data ? JSON.parse(data) : null
}
```

#### JSON 파일 (간단한 데이터)

```javascript
// scripts/update-json-data.js
const fs = require('fs')
const path = require('path')

function updateBestsellersData(newData) {
  const filePath = path.join(process.cwd(), 'data', 'bestsellers.json')
  fs.writeFileSync(filePath, JSON.stringify(newData, null, 2))
  console.log('Bestsellers data updated')
}

module.exports = { updateBestsellersData }
```

## 5. 프론트엔드 구현

### 하이브리드 렌더링 전략

정적 생성(SSG)과 클라이언트 사이드 데이터 페칭을 조합:

```tsx
// pages/index.tsx
import { GetStaticProps } from 'next'
import { useState } from 'react'
import { useQuery } from 'react-query'
import BestsellerList from '../components/BestsellerList'
import { fetchBestsellers } from '../lib/api'

export default function HomePage({ initialData }) {
  // 초기 데이터는 SSG로 제공, 이후 최신 데이터 fetch
  const { data } = useQuery('bestsellers', fetchBestsellers, {
    initialData,
    refetchOnWindowFocus: false,
    refetchInterval: 3600000, // 1시간마다 갱신
  })

  const [bookstore, setBookstore] = useState('all')
  
  // 서점별 필터링 로직
  const filteredBooks = bookstore === 'all' 
    ? data 
    : data.filter(book => book.store === bookstore)

  return (
    <main className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-6">BookBite - 통합 베스트셀러</h1>
      
      <div className="flex space-x-2 mb-6">
        <button 
          className={`px-4 py-2 rounded ${bookstore === 'all' ? 'bg-blue-500 text-white' : 'bg-gray-200'}`}
          onClick={() => setBookstore('all')}
        >
          전체
        </button>
        <button 
          className={`px-4 py-2 rounded ${bookstore === 'yes24' ? 'bg-blue-500 text-white' : 'bg-gray-200'}`}
          onClick={() => setBookstore('yes24')}
        >
          예스24
        </button>
        {/* 다른 서점 버튼들... */}
      </div>
      
      <BestsellerList books={filteredBooks} />
    </main>
  )
}

export const getStaticProps: GetStaticProps = async () => {
  const bestsellers = await fetchBestsellers()
  return {
    props: { initialData: bestsellers },
    revalidate: 86400, // ISR: 24시간마다 재생성
  }
}
```

### 도서 상세 페이지

```tsx
// pages/book/[id].tsx
import { GetStaticPaths, GetStaticProps } from 'next'
import Image from 'next/image'
import { fetchBookById, fetchAllBookIds } from '../../lib/api'

export default function BookDetail({ book }) {
  if (!book) return <div>도서를 찾을 수 없습니다.</div>
  
  return (
    <div className="container mx-auto px-4 py-8">
      <div className="flex flex-col md:flex-row gap-8">
        <div className="w-full md:w-1/3">
          <div className="relative h-[400px] w-full">
            <Image 
              src={book.imageUrl || '/images/default-book.jpg'} 
              alt={book.title}
              fill
              className="object-contain"
            />
          </div>
        </div>
        
        <div className="w-full md:w-2/3">
          <h1 className="text-3xl font-bold mb-2">{book.title}</h1>
          <p className="text-lg mb-4">{book.author} 저 | {book.publisher}</p>
          <p className="text-xl font-bold text-red-500 mb-6">{book.price}원</p>
          
          <div className="flex space-x-4 mb-8">
            {book.storeLinks.map(link => (
              <a 
                key={link.store}
                href={link.url}
                target="_blank"
                rel="noopener noreferrer"
                className="px-6 py-3 bg-blue-500 text-white rounded hover:bg-blue-600 transition"
              >
                {link.store} 구매하기
              </a>
            ))}
          </div>
          
          <div className="prose max-w-none">
            <h2>도서 소개</h2>
            <div dangerouslySetInnerHTML={{ __html: book.description }} />
          </div>
        </div>
      </div>
    </div>
  )
}

export const getStaticPaths: GetStaticPaths = async () => {
  const ids = await fetchAllBookIds()
  return {
    paths: ids.map(id => ({ params: { id } })),
    fallback: 'blocking'
  }
}

export const getStaticProps: GetStaticProps = async ({ params }) => {
  const book = await fetchBookById(params.id)
  if (!book) {
    return { notFound: true }
  }
  return {
    props: { book },
    revalidate: 86400 // 24시간마다 재생성
  }
}
```

## 6. 관리자 인터페이스

### 데이터 수집 페이지

```tsx
// pages/admin/data-collection.tsx
import { useState } from 'react'
import { useMutation } from 'react-query'
import { triggerDataCollection } from '../../lib/api'

export default function DataCollectionPage() {
  const [collectionType, setCollectionType] = useState('bestseller')
  const [keyword, setKeyword] = useState('')
  const [maxItems, setMaxItems] = useState(50)
  
  const mutation = useMutation(triggerDataCollection)
  
  const handleSubmit = async (e) => {
    e.preventDefault()
    mutation.mutate({
      type: collectionType,
      keyword,
      maxItems
    })
  }
  
  return (
    <div className="max-w-4xl mx-auto p-6">
      <h1 className="text-2xl font-bold mb-6">데이터 수집</h1>
      
      <form onSubmit={handleSubmit} className="space-y-6">
        <div>
          <label className="block mb-2">수집 유형</label>
          <select 
            value={collectionType} 
            onChange={(e) => setCollectionType(e.target.value)}
            className="w-full p-2 border rounded"
          >
            <option value="bestseller">베스트셀러</option>
            <option value="review">도서 리뷰</option>
            <option value="youtube">유튜브 영상</option>
            <option value="news">도서 뉴스</option>
          </select>
        </div>
        
        <div>
          <label className="block mb-2">검색 키워드 (선택)</label>
          <input
            type="text"
            value={keyword}
            onChange={(e) => setKeyword(e.target.value)}
            className="w-full p-2 border rounded"
          />
        </div>
        
        <div>
          <label className="block mb-2">최대 수집 항목</label>
          <input
            type="number"
            min="10"
            max="100"
            value={maxItems}
            onChange={(e) => setMaxItems(Number(e.target.value))}
            className="w-full p-2 border rounded"
          />
        </div>
        
        <button 
          type="submit" 
          disabled={mutation.isLoading}
          className="px-6 py-3 bg-blue-500 text-white rounded hover:bg-blue-600 disabled:bg-gray-400"
        >
          {mutation.isLoading ? '수집 중...' : '수집 시작'}
        </button>
      </form>
      
      {mutation.isSuccess && (
        <div className="mt-6 p-4 bg-green-100 text-green-700 rounded">
          데이터 수집이 성공적으로 완료되었습니다.
          <p>수집된 항목: {mutation.data.itemCount}개</p>
        </div>
      )}
      
      {mutation.isError && (
        <div className="mt-6 p-4 bg-red-100 text-red-700 rounded">
          오류가 발생했습니다: {mutation.error.message}
        </div>
      )}
    </div>
  )
}
```

### 데이터 수집 API 엔드포인트

```typescript
// pages/api/admin/collect-data.ts
import type { NextApiRequest, NextApiResponse } from 'next'
import { collectBestsellers, collectReviews, collectYoutubeVideos, collectNews } from '../../../lib/collectors'

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  // 관리자 인증 확인
  // ...인증 로직...
  
  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method not allowed' })
  }
  
  const { type, keyword, maxItems } = req.body
  
  try {
    let result
    
    switch (type) {
      case 'bestseller':
        result = await collectBestsellers(maxItems)
        break
      case 'review':
        result = await collectReviews(keyword, maxItems)
        break
      case 'youtube':
        result = await collectYoutubeVideos(keyword, maxItems)
        break
      case 'news':
        result = await collectNews(keyword, maxItems)
        break
      default:
        return res.status(400).json({ error: 'Invalid collection type' })
    }
    
    res.status(200).json({
      success: true,
      itemCount: result.length,
      sample: result.slice(0, 3)
    })
  } catch (error) {
    res.status(500).json({ 
      error: 'Collection failed',
      message: error.message
    })
  }
}
```

## 7. 배포 및 운영 전략

### Vercel 배포 설정

1. GitHub 저장소와 Vercel 연결
2. 환경 변수 설정 (API 키, 데이터베이스 연결 정보 등)
3. 빌드 및 배포 설정 구성
4. 커스텀 도메인 연결 (선택 사항)

### 배포 자동화

- **개발 브랜치**: 기능 개발 및 버그 수정
- **스테이징 브랜치**: 테스트 및 QA
- **메인 브랜치**: 운영 환경 배포

### 모니터링 및 로깅

- **Vercel Analytics**: 사용자 행동 및 성능 분석
- **Sentry**: 오류 모니터링 및 알림
- **Logtail/Datadog**: 로그 집계 및 분석

## 8. 마이그레이션 로드맵

### 단계적 전환 계획

#### 1단계: 기초 설정 (1주)
- GitHub 저장소 설정
- Next.js 프로젝트 생성
- Vercel 연결 및 기본 배포

#### 2단계: 데이터 수집 모듈 구현 (2주)
- 베스트셀러 수집 스크립트 구현
- 도서 리뷰, 유튜브, 뉴스 수집기 구현
- GitHub Actions 스케줄링 설정

#### 3단계: 백엔드 API 구현 (2주)
- 데이터 모델 설계
- API 엔드포인트 구현
- 데이터베이스 연결 구축

#### 4단계: 프론트엔드 구현 (3주)
- 메인 페이지 구현
- 베스트셀러 목록 및 상세 페이지
- 서점별 필터링 기능

#### 5단계: 관리자 인터페이스 구현 (1주)
- 데이터 수집 관리 페이지
- 콘텐츠 관리 기능

#### 6단계: 테스트 및 최적화 (1주)
- 성능 최적화
- 크로스 브라우저 테스트
- 모바일 대응성 확인

#### 7단계: 배포 및 전환 (1주)
- 최종 테스트
- 운영 환경 배포
- 도메인 전환
- 모니터링 시스템 구축

### 개발 우선순위

1. 베스트셀러 수집 및 표시 기능
2. 서점별 필터링
3. 도서 상세 정보 및 링크
4. 관리자 데이터 수집 인터페이스
5. 추가 데이터 수집 (리뷰, 유튜브, 뉴스)

## 9. 비용 분석

### Vercel 비용

- **Hobby 플랜 (무료)**
  - 개인 프로젝트에 적합
  - 100GB 대역폭/월
  - 서버리스 함수 제한적 실행
  - 프리뷰 배포 지원

- **Pro 플랜 ($20/월)**
  - 팀 협업 기능
  - 더 많은 대역폭 및 실행 시간
  - Cron Jobs 지원
  - 비밀 환경 변수 추가 제공

### 데이터베이스 비용

- **MongoDB Atlas (무료 ~ $57/월)**
  - 무료 티어: 512MB 스토리지
  - 공유 클러스터: $9/월부터
  - 전용 클러스터: $57/월부터

- **Vercel KV (무료 ~ $20/월)**
  - 무료 티어: 50MB 스토리지
  - Pro 티어: 256MB 스토리지 ($20/월)

### 총 예상 비용

- **초기 단계**: $0/월 (무료 티어)
- **확장 단계**: $29/월 (Vercel Pro + MongoDB Atlas 공유 클러스터)
- **성장 단계**: $77/월 (Vercel Pro + MongoDB Atlas 전용 클러스터)

## 10. 결론 및 기대효과

### GitHub + Vercel 전환의 장점

1. **안정성 향상**: 모던 아키텍처와 견고한 오류 처리로 안정적 서비스 제공
2. **성능 개선**: 정적 생성과 CDN을 통한 빠른 로딩 시간
3. **유지보수 용이성**: 체계적인 코드 구조와 자동화된 테스트
4. **확장성**: 트래픽 증가에도 유연하게 대응 가능
5. **개발 생산성**: 최신 도구와 워크플로우로 빠른 기능 개발

### 새롭게 추가 가능한 기능

1. **개인화 추천**: 사용자 행동 기반 도서 추천
2. **독서 트렌드 분석**: 장르 및 주제별 트렌드 시각화
3. **SNS 연동**: 도서 정보 공유 기능
4. **리뷰 감성 분석**: AI를 활용한 리뷰 분석
5. **푸시 알림**: 관심 도서 관련 소식 알림

이 전환은 초기에 개발 노력이 필요하지만, 장기적으로 더 안정적이고 확장 가능한 서비스를 제공할 수 있는 기반이 될 것입니다. GitHub와 Vercel의 조합은 개발 생산성과 배포 효율성을 크게 향상시켜, BookBite가 더 나은 사용자 경험을 제공하고 새로운 기능을 빠르게 도입할 수 있도록 합니다. 