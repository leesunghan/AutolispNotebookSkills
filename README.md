# AutoLISP Notebook (`.lspnb`) Agent Skills

> [!NOTE]
> 본 저장소는 **AutoLISP Notebook (`.lspnb`)** 파일을 AI 코딩 에이전트(Claude Code, Cursor, Copilot, Antigravity 등)가 빌드, 검수 및 유지 관리할 수 있도록 설계된 전문적인 스킬(Skills) 및 페르소나(Personas) 모음입니다.
> 
> 이 프로젝트는 Addy Osmani 님이 공개한 [agent-skills](https://github.com/addyosmani/agent-skills) 저장소를 기반으로 커스텀 편집되었습니다.

---

## 1. AutoLISP Notebook 소개

**AutoLISP Notebook**은 AutoCAD 및 BricsCAD 환경을 위한 대화형(Interactive) 개발 및 교육용 프레임워크입니다. Jupyter Notebook과 유사한 사용자 경험을 제공하여 CAD 자동화를 대중화하고 개발 편의성을 극대화합니다.

*   **통합형 문서 포맷 (`.lspnb`)**: 설명(Markdown 셀)과 실행 가능한 코드(AutoLISP/DCL 셀)가 교대로 교차하도록 구성되어 있습니다. 사용자는 직관적인 문서를 읽으며 코드를 즉각적으로 실행하고 CAD 상에서 실시간 반응을 확인할 수 있습니다.
*   **VS Code 확장 도구 (VS Code Extension)**: `.lspnb` 파일을 VS Code 에디터 내에서 셀 단위로 렌더링하고 직관적으로 편집할 수 있는 확장 프로그램입니다.
*   **캐드 애드온 (CAD Add-on / Connector)**: VS Code와 활성화된 CAD 런타임(AutoCAD 또는 BricsCAD) 프로세스 간의 통신 브릿지 역할을 담당합니다. VS Code 노트북에서 실행한 AutoLISP/DCL 코드가 해당 커넥터를 통해 CAD 상에서 즉시 실행됩니다.

---

## 2. 설치 및 설정 (Installation & Setup)

AutoLISP Notebook 환경을 구축하려면 다음 단계를 완료하십시오.

### 2.1 VS Code 확장 도구 설치
1. **VS Code**를 실행합니다.
2. 확장 뷰 (`Ctrl + Shift + X`)를 엽니다.
3. 마켓플레이스 검색창에 **`AutoLISP Notebook`**을 입력합니다.
4. 해당 확장을 찾아 **설치(Install)**를 클릭합니다.

### 2.2 캐드 애드온/커넥터 로드
1. **AutoCAD** 또는 **BricsCAD**를 실행합니다.
2. 캐드 명령행에 `APPLOAD` 명령을 입력하여 로드 대화상자를 엽니다.
3. 배포된 캐드 커넥터 파일(예: `LispNotebookListener.lsp` 또는 커넥터 모듈)을 선택하여 로드합니다.
4. 로드가 완료되면 캐드와 VS Code 간의 대화형 원격 실행 세션이 활성화됩니다.
5. VS Code 노트북의 우측 상단 커널 선택에서 타겟 캐드 환경(`AutoLISP`)을 선택하고 실행을 확인합니다.

---

## 3. 에이전트 오케스트레이션 및 프로젝트 디렉터리 구성

본 저장소는 개발 에이전트가 완벽한 규칙 하에 노트북 파일을 가공할 수 있도록 돕는 세부 사양과 프로젝트 격리 환경을 제공합니다.

*   **프로젝트 격리 (`projects/<프로젝트명>/`)**: 루트 디렉터리의 복잡성을 방지하고 개별 자동화 과제를 격리 보존하기 위해, 모든 신규 노트북(`.lspnb`) 및 계획 관련 산출물(`SPEC.md`, `plan.md`, `todo.md`)은 해당 폴더 하위에 생성하고 관리해야 합니다.
*   **스킬 (Skills)** (`/skills`): 사양 정의, 작업 분할, 점진적 구현, TDD 검증, 디버깅, 코드 품질 리뷰, 배포, 그리고 **DCL 기반 프론트엔드 UI 엔지니어링**에 이르는 단계별 엔지니어링 스킬 워크플로우 세트.
*   **페르소나 (Personas)** (`/agents`): 에이전트가 역할을 나누어 코드 리뷰, 보안 감사, 테스트 및 성능 최적화를 전담하도록 돕는 역할 가이드.
*   **가이드라인 (`AGENTS.md`)**: 에이전트가 노트북을 다룰 때 지켜야 할 핵심 규칙과 프로젝트 하위 경로 규칙, 게이트 제어 워크플로우 수록.

---

## 4. 개발사 및 라이선스 (Developer & License)

*   **개발사 및 원작자 (Developer & Creator)**:
    *   Copyright (c) 2025 
    *   본 저장소의 에이전트 스킬 워크플로우 프레임워크는 Addy Osmani의 [`agent-skills`](https://github.com/addyosmani/agent-skills)를 기반으로 하여, AutoLISP Notebook 설계/구현 표준에 맞게 특화 및 한국어로 정립되었습니다.
*   **라이선스 (License)**:
    *   본 프로젝트는 **[MIT License](file:///d:/Projects/SkillsProjects/AutolispNotebookSkills/LICENSE.txt)** 하에 제공되는 오픈소스 소프트웨어입니다.
