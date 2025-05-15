<p align="center">
  <img src="https://github.com/user-attachments/assets/57d23950-206b-4640-a649-66a175660ade" alt="Shortest logo" width="128" />
</p>

# Shortest

AI-powered natural language end-to-end testing framework.

[한국어](#한국어)

<video src="https://github.com/user-attachments/assets/d443279e-7364-452b-9f50-0c8dd0cf55fc" controls autoplay loop muted>
Your browser does not support the video tag.
</video>

## Features

- Natural language E2E testing framework
- AI-powered test execution using Anthropic Claude API
- Built on Playwright
- GitHub integration with 2FA support
- Email validation with Mailosaur

## 한국어

# Shortest

AI 기반 자연어 엔드투엔드 테스트 프레임워크입니다.

## 주요 기능

- 자연어 E2E 테스트 프레임워크
- Anthropic Claude API를 활용한 AI 기반 테스트 실행
- Playwright 기반 구축
- 2FA 지원 GitHub 통합
- Mailosaur를 통한 이메일 검증

## 프로젝트에서 Shortest 사용하기

### 설치

새 프로젝트나 기존 프로젝트에서 `shortest init` 명령어를 사용하여 설정 과정을 간소화할 수 있습니다.

`shortest init` 명령어는 다음과 같은 작업을 수행합니다:

```sh
npx @antiwork/shortest init
```

이 명령어는 다음을 수행합니다:

- `@antiwork/shortest` 패키지를 dev 의존성으로 자동 설치 (이미 설치되어 있지 않은 경우)
- 기본 `shortest.config.ts` 파일과 보일러플레이트 설정 생성
- 필요한 환경 변수(예: `ANTHROPIC_API_KEY`)에 대한 플레이스홀더가 포함된 `.env.local` 파일 생성 (이미 존재하지 않는 경우)
- `.env.local`과 `.shortest/`를 `.gitignore`에 추가

### 빠른 시작

1. 테스트 진입점을 결정하고 `shortest.config.ts` 설정 파일에 Anthropic API 키를 추가합니다:

```typescript
import type { ShortestConfig } from "@antiwork/shortest";

export default {
  headless: false,
  baseUrl: "http://localhost:3000",
  browser: {
    contextOptions: {
      ignoreHTTPSErrors: true
    },
  },
  testPattern: "**/*.test.ts",
  ai: {
    provider: "anthropic",
  },
} satisfies ShortestConfig;
```

Anthropic API 키는 기본적으로 `SHORTEST_ANTHROPIC_API_KEY` / `ANTHROPIC_API_KEY` 환경 변수를 사용합니다. `ai.config.apiKey`를 통해 재정의할 수 있습니다.

선택적으로 `browser.contextOptions` 속성을 사용하여 브라우저 동작을 구성할 수 있습니다. 이를 통해 [Playwright 브라우저 컨텍스트 옵션](https://playwright.dev/docs/api/class-browser#browser-new-context)을 전달할 수 있습니다.

2. 설정에서 지정한 패턴을 사용하여 테스트 파일을 생성합니다: `app/login.test.ts`

```typescript
import { shortest } from "@antiwork/shortest";

shortest("이메일과 비밀번호를 사용하여 앱에 로그인", {
  username: process.env.GITHUB_USERNAME,
  password: process.env.GITHUB_PASSWORD,
});
```

### 콜백 함수 사용

콜백 함수를 사용하여 추가적인 검증과 로직을 추가할 수 있습니다. AI는 브라우저에서 테스트 실행이 완료된 후 콜백 함수를 실행합니다.

```typescript
import { shortest } from "@antiwork/shortest";
import { db } from "@/lib/db/drizzle";
import { users } from "@/lib/db/schema";
import { eq } from "drizzle-orm";

shortest("사용자 이름과 비밀번호로 앱에 로그인", {
  username: process.env.USERNAME,
  password: process.env.PASSWORD,
}).after(async ({ page }) => {
  // 페이지에서 현재 사용자의 clerk ID 가져오기
  const clerkId = await page.evaluate(() => {
    return window.localStorage.getItem("clerk-user");
  });

  if (!clerkId) {
    throw new Error("데이터베이스에서 사용자를 찾을 수 없습니다");
  }

  // 데이터베이스 쿼리
  const [user] = await db
    .select()
    .from(users)
    .where(eq(users.clerkId, clerkId))
    .limit(1);

  expect(user).toBeDefined();
});
```

### 라이프사이클 훅

테스트 전후에 코드를 실행하기 위해 라이프사이클 훅을 사용할 수 있습니다.

```typescript
import { shortest } from "@antiwork/shortest";

shortest.beforeAll(async ({ page }) => {
  await clerkSetup({
    frontendApiUrl:
      process.env.PLAYWRIGHT_TEST_BASE_URL ?? "http://localhost:3000",
  });
});

shortest.beforeEach(async ({ page }) => {
  await clerk.signIn({
    page,
    signInParams: {
      strategy: "email_code",
      identifier: "iffy+clerk_test@example.com",
    },
  });
});

shortest.afterEach(async ({ page }) => {
  await page.close();
});

shortest.afterAll(async ({ page }) => {
  await clerk.signOut({ page });
});
```

### 테스트 체이닝

Shortest는 유연한 테스트 체이닝 패턴을 지원합니다:

```typescript
// 순차적 테스트 체인
shortest([
  "사용자가 이메일과 비밀번호로 로그인할 수 있음",
  "사용자가 계정 수준의 환불 정책을 수정할 수 있음",
]);

// 재사용 가능한 테스트 흐름
const loginAsLawyer = "유효한 자격 증명으로 변호사로 로그인";
const loginAsContractor = "유효한 자격 증명으로 계약자로 로그인";
const allAppActions = ["회사에 청구서 보내기", "청구서 보기"];

// 스프레드 연산자로 흐름 결합
shortest([loginAsLawyer, ...allAppActions]);
shortest([loginAsContractor, ...allAppActions]);
```

### API 테스트

자연어를 사용하여 API 엔드포인트 테스트

```typescript
const req = new APIRequest({
  baseURL: API_BASE_URI,
});

shortest(
  "응답에 활성 사용자만 포함되어 있는지 확인",
  req.fetch({
    url: "/users",
    method: "GET",
    params: new URLSearchParams({
      active: true,
    }),
  }),
);
```

또는 간단히:

```typescript
shortest(`
  ${API_BASE_URI}/users API GET 엔드포인트를 쿼리 파라미터 { "active": true }로 테스트
  응답에 활성 사용자만 포함되어 있는지 확인
`);
```

### 테스트 실행

```bash
pnpm shortest                   # 모든 테스트 실행
pnpm shortest login.test.ts     # 특정 파일의 테스트 실행
pnpm shortest login.test.ts:23  # 파일의 특정 테스트를 라인 번호로 실행
pnpm shortest --headless        # 헤드리스 모드로 실행
```

`examples` 디렉토리에서 예제 테스트를 찾을 수 있습니다.

### CI 설정

헤드리스 모드로 테스트를 실행하여 CI/CD 파이프라인에서 Shortest를 실행할 수 있습니다. CI/CD 파이프라인 시크릿에 Anthropic API 키를 추가해야 합니다.

### GitHub 2FA 로그인 설정

Shortest는 GitHub 2FA를 사용한 로그인을 지원합니다. GitHub 인증 테스트를 위해:

1. 저장소 설정으로 이동
2. "Password and Authentication"으로 이동
3. "Authenticator App" 클릭
4. "Use your authenticator app" 선택
5. "Setup key"를 클릭하여 OTP 시크릿 획득
6. OTP 시크릿을 `.env.local` 파일에 추가하거나 Shortest CLI를 사용하여 추가
7. 터미널에 표시된 2FA 코드를 Github의 Authenticator 설정 페이지에 입력하여 프로세스 완료

```bash
shortest --github-code --secret=<OTP_SECRET>
```

### 환경 설정

`.env.local`에 필요한 항목:

```bash
ANTHROPIC_API_KEY=your_api_key
GITHUB_TOTP_SECRET=your_secret  # GitHub 인증 테스트에만 필요
```

## Shortest CLI development

The [NPM package](https://www.npmjs.com/package/@antiwork/shortest) is located in [`packages/shortest/`](./packages/shortest). See [CONTRIBUTING](./packages/shortest/CONTRIBUTING.md) guide.

## Web app development

This guide will help you set up the Shortest web app for local development.

### Prerequisites

- React >=19.0.0 (if using with Next.js 14+ or Server Actions)
- Next.js >=14.0.0 (if using Server Components/Actions)

> [!WARNING]
> Using this package with React 18 in Next.js 14+ projects may cause type conflicts with Server Actions and `useFormStatus`
>
> If you encounter type errors with form actions or React hooks, ensure you're using React 19

### Getting started

1. Clone the repository:

```bash
git clone https://github.com/antiwork/shortest.git
cd shortest
```

2. Install dependencies:

```bash
npm install -g pnpm
pnpm install
```

### Environment setup

#### For Antiwork team members

Pull Vercel env vars:

```bash
pnpm i -g vercel
vercel link
vercel env pull
```

#### For other contributors

1. Run `pnpm run setup` to configure the environment variables.
2. The setup wizard will ask you for information. Refer to "Services Configuration" section below for more details.

### Set up the database

```bash
pnpm drizzle-kit generate
pnpm db:migrate
pnpm db:seed # creates stripe products, currently unused
```

### Services configuration

You'll need to set up the following services for local development. If you're not an Antiwork Vercel team member, you'll need to either run the setup wizard `pnpm run setup` or manually configure each of these services and add the corresponding environment variables to your `.env.local` file:

<details>
<summary>Clerk</summary>

1. Go to [clerk.com](https://clerk.com) and create a new app.
2. Name it whatever you like and **disable all login methods except GitHub**.
   ![Clerk App Login](https://github.com/user-attachments/assets/1de7aebc-8e9d-431a-ae13-af60635307a1)
3. Once created, copy the environment variables to your `.env.local` file.
   ![Clerk Env Variables](https://github.com/user-attachments/assets/df3381e6-017a-4e01-8bd3-5793e5f5d31e)
4. In the Clerk dashboard, disable the "Require the same device and browser" setting to ensure tests with Mailosaur work properly.

</details>

<details>
<summary>Vercel Postgres</summary>

1. Go to your dashboard at [vercel.com](https://vercel.com).
2. Navigate to the Storage tab and click the `Create Database` button.
   ![Vercel Create Database](https://github.com/user-attachments/assets/acdf3ba7-31a6-498b-860c-171018d5ba02)
3. Choose `Postgres` from the `Browse Storage` menu.
   ![Neon Postgres](https://github.com/user-attachments/assets/9ad2a391-5213-4f31-a6c3-b9e54c69bb2e)
4. Copy your environment variables from the `Quickstart` `.env.local` tab.
   ![Vercel Postgres .env.local](https://github.com/user-attachments/assets/e48f1d96-2fd6-4e2e-aaa6-eeb5922cc521)

</details>

<details>
<summary>Anthropic</summary>

1. Go to your dashboard at [anthropic.com](https://anthropic.com) and grab your API Key.
   - Note: If you've never done this before, you will need to answer some questions and likely load your account with a balance. Not much is needed to test the app.
     ![Anthropic API Key](https://github.com/user-attachments/assets/0905ed4b-5815-4d50-bf43-8713a4397674)

</details>

<details>
<summary>Stripe</summary>

1. Go to your `Developers` dashboard at [stripe.com](https://stripe.com).
2. Turn on `Test mode`.
3. Go to the `API Keys` tab and copy your `Secret key`.
   ![Stripe Secret Key](https://github.com/user-attachments/assets/0830b226-f2c2-4b92-a28f-f4682ad03ec0)
4. Go to the terminal of your project and type `pnpm run stripe:webhooks`. It will prompt you to login with a code then give you your `STRIPE_WEBHOOK_SECRET`.
   ![Stripe Webhook Secret](https://github.com/user-attachments/assets/b02531ed-5c31-40ba-8483-32880aa3ca36)

</details>

<details>
<summary>GitHub OAuth</summary>

1. Create a GitHub OAuth App:

   - Go to your GitHub account settings.
   - Navigate to `Developer settings` > `OAuth Apps` > `New OAuth App`.
   - Fill in the application details:
     - **Application name**: Choose any name for your app
     - **Homepage URL**: Set to `http://localhost:3000` for local development
     - **Authorization callback URL**: Use the Clerk-provided callback URL (found in below image)
       ![Github OAuth App](https://github.com/user-attachments/assets/1af635fd-dedc-401c-a45a-159cb20bb209)

2. Configure Clerk with GitHub OAuth:
   - Go to your Clerk dashboard.
   - Navigate to `Configure` > `SSO Connections` > `GitHub`.
   - Select `Use custom credentials`
   - Enter your `Client ID` and `Client Secret` from the GitHub OAuth app you just created.
   - Add `repo` to the `Scopes`
     ![Clerk Custom Credentials](https://github.com/user-attachments/assets/31d414e1-4e1e-4725-8649-ec1826c6e53e)

</details>

<details>
<summary>Mailosaur</summary>

1. [Sign up](https://mailosaur.com/app/signup) for an account with Mailosaur.
2. Create a new Inbox/Server.
3. Go to [API Keys](https://mailosaur.com/app/keys) and create a standard key.
4. Update the environment variables:
   - `MAILOSAUR_API_KEY`: Your API key
   - `MAILOSAUR_SERVER_ID`: Your server ID

The email used to test the login flow will have the format `shortest@<MAILOSAUR_SERVER_ID>.mailosaur.net`, where
`MAILOSAUR_SERVER_ID` is your server ID.
Make sure to add the email as a new user under the Clerk app.

</details>

### Running locally

Run the development server:

```bash
pnpm dev
```

Open [http://localhost:3000](http://localhost:3000) in your browser to see the app in action.
