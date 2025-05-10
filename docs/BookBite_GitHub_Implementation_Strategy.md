# BookBite GitHub 기반 구현 전략

## 1. 프로젝트 분석

### 현재 BookBite 사이트 목표
- **베스트셀러 통합 플랫폼**: 한국 주요 인터넷 서점(예스24, 알라딘, 교보문고)의 베스트셀러 정보를 한 곳에서 제공
- **도서 정보 중개**: 사용자에게 도서 정보를 제공하고 해당 서점으로 연결
- **수익 모델**: 제휴 마케팅을 통한 커미션 수익 창출
- **데이터 수집**: 각 서점에서 베스트셀러, 신간, 도서 뉴스, 관련 유튜브 콘텐츠 수집

### 주요 기능 요구사항
1. 베스트셀러 순위 정보 통합 제공
2. 도서 상세 정보 제공 (표지, 저자, 출판사, 가격 등)
3. 서점별 바로가기 링크 제공
4. 주기적인 데이터 수집 및 업데이트
5. 관리자용 데이터 수집 인터페이스
6. 사용자 친화적 UI/UX 제공

## 2. GitHub 기반 재구현 전략

### 기술 스택 제안

#### 프론트엔드
- **Next.js**: React 기반 프레임워크로 서버 사이드 렌더링(SSR) 및 정적 사이트 생성(SSG) 지원
- **TypeScript**: 타입 안정성 확보 및 유지보수성 향상
- **Tailwind CSS**: 반응형 디자인을 위한 유틸리티 우선 CSS 프레임워크
- **Redux Toolkit**: 상태 관리 (필요시)
- **React Query**: 서버 상태 관리 및 데이터 fetching

#### 백엔드
- **Fastify/NestJS**: 효율적인 API 서버 구현
- **Prisma**: 타입 안전한, 최신 ORM을 통한 데이터베이스 상호작용
- **PostgreSQL/MongoDB**: 도서 데이터 저장용 데이터베이스
- **Redis**: 캐싱 솔루션 (베스트셀러 정보 캐싱)

#### 인프라
- **Docker**: 컨테이너화된 개발 및 배포 환경
- **GitHub Actions**: CI/CD 파이프라인 자동화
- **Vercel/Netlify**: 프론트엔드 배포
- **Heroku/Railway/AWS**: 백엔드 서비스 배포

#### 데이터 수집
- **Node.js 스크립트**: 크롤링 및 API 수집 자동화
- **Cheerio/Puppeteer**: 웹 크롤링
- **Node-cron**: 스케줄링
- **YouTube Data API**: 유튜브 영상 데이터 수집
- **News API**: 도서 관련 뉴스 수집

### 레포지토리 구조

```
bookbite/
├── .github/                 # GitHub Actions 워크플로우
├── packages/                # 모노레포 구조
│   ├── frontend/            # Next.js 프론트엔드
│   ├── backend/             # API 서버
│   ├── data-collectors/     # 데이터 수집 모듈
│   ├── database/            # 데이터베이스 스키마 및 마이그레이션
│   └── shared/              # 공유 타입, 유틸리티 등
├── config/                  # 설정 파일
├── docs/                    # 프로젝트 문서
└── .env.example             # 환경 변수 예시
```

## 3. 데이터 수집 모듈 설계

### 모던 크롤링 아키텍처
기존 도서 수집 방식을 개선하여 더 안정적이고 유지보수하기 쉬운 구조로 재설계:

```typescript
// packages/data-collectors/src/bookstores/yes24.ts
import { BookData } from '../types';
import { fetchHTML, parsePrice, logError, saveHTML } from '../utils';

// 다양한 선택자 패턴 준비 (기존 전략 유지)
const SELECTORS = {
  item: [
    '//div[contains(@class, "goodsTxtInfo")]',
    '//div[contains(@class, "item_info")]',
    '//ul[contains(@class, "bestSellerList")]/li',
  ],
  title: [
    './/a[@class="gd_name"]',
    './/a[contains(@class, "item_title")]',
  ],
  // ... 기타 필드 선택자
};

export async function collectYes24Bestsellers(): Promise<BookData[]> {
  try {
    const url = 'http://www.yes24.com/24/category/bestseller';
    const html = await fetchHTML(url, {
      headers: { 
        'User-Agent': 'Mozilla/5.0 ...',
        // ... 기타 헤더
      }
    });
    
    // HTML 샘플 저장 (구조 변경 감지용)
    await saveHTML('yes24', html);
    
    // 데이터 파싱 로직
    const books = parseYes24HTML(html);
    
    return books;
  } catch (error) {
    logError('yes24_collector', error);
    return [];
  }
}
```

### 스케줄링 및 API 통합
GitHub Actions와 연동하여 자동화된 데이터 수집 파이프라인 구축:

```yaml
# .github/workflows/collect-data.yml
name: Collect Bookstore Data

on:
  schedule:
    - cron: '0 0 * * *'  # 매일 자정 실행
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
        run: cd packages/data-collectors && npm install
      - name: Run collectors
        run: cd packages/data-collectors && npm run collect
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
          YOUTUBE_API_KEY: ${{ secrets.YOUTUBE_API_KEY }}
      - name: Update API data
        run: cd packages/backend && npm run update-cache
```

## 4. 프론트엔드 구현

### 정적 생성 + 동적 업데이트 하이브리드 접근법

Next.js를 활용한 하이브리드 렌더링 전략:

1. **정적 페이지 생성 (SSG)**:
   - 홈페이지 및 베스트셀러 목록 페이지는 빌드 시 정적 생성
   - 데이터 수집 후 자동 재빌드 트리거 (GitHub Actions)

2. **클라이언트 측 데이터 갱신**:
   - 최신 데이터를 API로 제공하여 클라이언트에서 필요시 갱신
   - SWR/React Query를 활용한 주기적 데이터 폴링

```tsx
// packages/frontend/pages/index.tsx
import { GetStaticProps } from 'next';
import { useState } from 'react';
import { useQuery } from 'react-query';
import BestsellerList from '../components/BestsellerList';
import { fetchBestsellers } from '../api/bestsellers';

export default function HomePage({ initialData }) {
  // 초기 데이터는 SSG로 제공, 이후 최신 데이터 fetch
  const { data } = useQuery('bestsellers', fetchBestsellers, {
    initialData,
    refetchOnWindowFocus: false,
    refetchInterval: 3600000, // 1시간마다 갱신
  });

  const [bookstore, setBookstore] = useState('all');
  
  // 서점별 필터링 로직
  const filteredBooks = bookstore === 'all' 
    ? data 
    : data.filter(book => book.store === bookstore);

  return (
    <main>
      <h1>BookBite - 통합 베스트셀러</h1>
      
      <div className="filters">
        <button onClick={() => setBookstore('all')}>전체</button>
        <button onClick={() => setBookstore('yes24')}>예스24</button>
        <button onClick={() => setBookstore('aladin')}>알라딘</button>
        <button onClick={() => setBookstore('kyobo')}>교보문고</button>
      </div>
      
      <BestsellerList books={filteredBooks} />
    </main>
  );
}

export const getStaticProps: GetStaticProps = async () => {
  const bestsellers = await fetchBestsellers();
  return {
    props: { initialData: bestsellers },
    revalidate: 86400, // ISR: 24시간마다 재생성
  };
};
```

## 5. 관리자 인터페이스

기존의 워드프레스 관리자 인터페이스를 대체할 독립적인 관리자 페이지:

```tsx
// packages/frontend/pages/admin/data-collection.tsx
import { useState } from 'react';
import { useQuery, useMutation } from 'react-query';
import { triggerDataCollection } from '../../api/admin';

export default function DataCollectionPage() {
  const [collectionType, setCollectionType] = useState('bestseller');
  const [keyword, setKeyword] = useState('');
  const [maxItems, setMaxItems] = useState(50);
  
  const mutation = useMutation(triggerDataCollection);
  
  const handleSubmit = async (e) => {
    e.preventDefault();
    mutation.mutate({
      type: collectionType,
      keyword,
      maxItems
    });
  };
  
  return (
    <div className="admin-page">
      <h1>데이터 수집</h1>
      
      <form onSubmit={handleSubmit}>
        <div>
          <label>수집 유형</label>
          <select 
            value={collectionType} 
            onChange={(e) => setCollectionType(e.target.value)}
          >
            <option value="bestseller">베스트셀러</option>
            <option value="review">도서 리뷰</option>
            <option value="youtube">유튜브 영상</option>
            <option value="news">도서 뉴스</option>
          </select>
        </div>
        
        <div>
          <label>검색 키워드 (선택)</label>
          <input
            type="text"
            value={keyword}
            onChange={(e) => setKeyword(e.target.value)}
          />
        </div>
        
        <div>
          <label>최대 수집 항목</label>
          <input
            type="number"
            min="10"
            max="100"
            value={maxItems}
            onChange={(e) => setMaxItems(Number(e.target.value))}
          />
        </div>
        
        <button 
          type="submit" 
          disabled={mutation.isLoading}
        >
          {mutation.isLoading ? '수집 중...' : '수집 시작'}
        </button>
      </form>
      
      {mutation.isSuccess && (
        <div className="success-message">
          데이터 수집이 성공적으로 완료되었습니다.
          <p>수집된 항목: {mutation.data.itemCount}개</p>
        </div>
      )}
      
      {mutation.isError && (
        <div className="error-message">
          오류가 발생했습니다: {mutation.error.message}
        </div>
      )}
    </div>
  );
}
```

## 6. GitHub 기반 협업 및 배포 전략

### CI/CD 파이프라인
- **코드 품질 관리**: ESLint, Prettier, TypeScript 타입 체크
- **자동화된 테스트**: Jest, React Testing Library
- **배포 자동화**: 
  - Pull Request 빌드 검증
  - 메인 브랜치 병합 시 자동 배포
  - 데이터 수집 후 자동 재배포 (최신 데이터 반영)

### 개발 워크플로우
1. 기능 브랜치에서 개발
2. Pull Request 생성 및 코드 리뷰
3. 자동화된 테스트 통과 후 메인 브랜치에 병합
4. GitHub Actions 통한 자동 배포

### 배포 환경 설정
- **개발 환경**: PR 미리보기 배포 (Vercel/Netlify)
- **스테이징 환경**: 테스트 데이터로 모든 기능 검증
- **운영 환경**: 실제 데이터 및 운영용 설정

## 7. 마이그레이션 로드맵

### 단계적 전환 계획
1. **초기 설정 (1주차)**
   - GitHub 저장소 설정
   - 프로젝트 구조 잡기
   - 기본 Next.js 앱 구현

2. **데이터 수집 모듈 구현 (2-3주차)**
   - 베스트셀러 수집 스크립트 최적화
   - 도서 리뷰, 유튜브, 뉴스 수집기 구현
   - GitHub Actions 스케줄링 설정

3. **API 서버 구현 (3-4주차)**
   - 데이터 모델 및 스키마 설계
   - CRUD API 엔드포인트 구현
   - 관리자 API 구현

4. **프론트엔드 구현 (4-6주차)**
   - 메인 페이지 및 베스트셀러 목록
   - 도서 상세 페이지
   - 관리자 인터페이스

5. **테스트 및 최적화 (6-7주차)**
   - 단위 테스트 및 통합 테스트
   - 성능 최적화
   - SEO 최적화

6. **배포 및 전환 (8주차)**
   - 운영 환경 배포
   - DNS 전환
   - 모니터링 설정

## 8. 부가 가치 및 향상된 기능

### 기존 워드프레스 대비 GitHub 기반 개발의 장점
1. **빠른 로딩 속도**: 정적 페이지 및 최적화된 자바스크립트
2. **보안 향상**: PHP 취약점 제거 및 서버 공격 표면 감소
3. **비용 효율성**: 정적 페이지 호스팅은 대부분 무료 또는 저렴
4. **개발 생산성**: 최신 도구와 워크플로우로 빠른 기능 개발
5. **확장성**: 트래픽 증가에도 안정적인 성능

### 확장 가능한 새 기능
1. **도서 추천 시스템**: 사용자 행동 기반 추천
2. **리뷰 집계 및 분석**: 여러 소스의 리뷰 감성 분석
3. **독서 트렌드 시각화**: 인기 장르 및 트렌드 차트
4. **SNS 공유 기능**: 도서 정보 쉽게 공유
5. **개인화 도서 목록**: 관심 도서 저장 및 추적

## 9. 결론

GitHub 기반 개발로 전환함으로써 BookBite는 다음과 같은 이점을 얻을 수 있습니다:

1. **기술적 부채 해소**: 최신 기술 스택 도입으로 유지보수성 향상
2. **성능 개선**: 정적 사이트 생성 및 API 기반 아키텍처로 속도 향상
3. **안정적인 데이터 수집**: 자동화된 워크플로우로 지속적인 데이터 수집
4. **확장 가능성**: 새로운 기능 추가 및 확장이 용이한 구조
5. **개발 프로세스 개선**: 버전 관리, 코드 리뷰, 자동화된 테스트로 품질 향상

이 전환은 초기에 상당한 개발 노력이 필요하지만, 장기적으로 더 안정적이고 확장 가능한 서비스를 제공할 수 있는 기반이 될 것입니다. 