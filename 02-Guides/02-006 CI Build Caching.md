# How to configure Continuous Integration (CI) build caching
<span style="color: #808080;">Last updated April 22, 2025</span>

빌드 성능을 향상시키기 위해 Next.js는 빌드 간에 공유되는 캐시를 `.next/cache`에 저장합니다.

CI(지속적 통합) 환경에서 이 캐시를 활용하려면 빌드 간에 캐시를 올바르게 유지하도록 CI 워크플로를 구성해야 합니다.

> CI가 빌드 간에 `.next/cache`를 유지하도록 구성되지 않은 경우 [캐시가 감지되지 않음 오류](https://nextjs.org/docs/messages/no-cache)가 표시될 수 있습니다.

다음은 일반적인 CI 공급자에 대한 몇 가지 캐시 구성 예입니다:

## Vercel
Next.js 캐싱은 자동으로 구성됩니다. 귀하가 취해야 할 조치는 없습니다. Vercel에서 Turborepo를 사용하는 경우 [여기에서 자세히 알아보세요.](https://vercel.com/docs/monorepos/turborepo)

## CircleCI
`.next/cache`를 포함하도록 `.circleci/config.yml`의 `save_cache` 단계를 편집하세요:

```yaml
steps:
  - save_cache:
    key: dependency-cache{{ checksum "yarn.lock" }}
    paths:
      - ./node_modules
      - ./.next/cache
```

`save_cache` 키가 없으면 [빌드 캐싱 설정에 대한 CircleCI 설명서](https://circleci.com/docs/2.0/caching/)를 따르세요.

## Travis CI
`.travis.yml`에 다음을 추가하거나 병합하세요:
```yaml
cache:
  directories:
    - $HOME/.cache/yarn
    - node_modules
    - .next/cache
```

## GitLab CI
`.gitlab-ci.yml`에 다음을 추가하거나 병합하세요:
```yaml
cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - node_modules/
    - .next/cache/

```

## Netlify CI
[`@netlify/plugin-nextjs`](https://www.npmjs.com/package/@netlify/plugin-nextjs)와 함께 [Netlify 플러그인](https://www.netlify.com/products/build/plugins/)을 사용하세요.

## AWS CodeBuild
`buildspec.yml`에 다음을 추가(또는 병합)하세요:
```yaml
cache:
  paths:
    - 'node_modules/**/*' # 더 빠른 `yarn` 또는 `npm i`를 위해 `node_modules` 캐시
    - '.next/cache/**/*' # 더 빠른 애플리케이션 재구축을 위한 캐시 Next.js
```

## GitHub Actions
GitHub의 [actions/cache](https://github.com/actions/cache)를 사용하여 워크플로 파일에 다음 단계를 추가합니다.
```yaml
uses: actions/cache@v4
with:
  # `yarn`, `bun` 또는 기타 패키지 관리자 https://github.com/actions/cache/blob/main/examples.md를 사용한 캐싱에 대해서는 여기를 참조하거나 actions/setup-node https://github.com/actions/setup-node를 사용하여 캐싱을 활용할 수 있습니다.
  path: |
    ~/.npm
    ${{ github.workspace }}/.next/cache
  # 패키지나 소스 파일이 변경될 때마다 새 캐시를 생성합니다.
  key: ${{ runner.os }}-nextjs-${{ hashFiles('**/package-lock.json') }}-${{ hashFiles('**/*.js', '**/*.jsx', '**/*.ts', '**/*.tsx') }}
  # 소스 파일은 변경되었지만 패키지는 변경되지 않은 경우 이전 캐시에서 다시 빌드하세요.
  restore-keys: |
    ${{ runner.os }}-nextjs-${{ hashFiles('**/package-lock.json') }}-
```

## Bitbucket Pipelines
최상위 수준(`pipelines`과 동일한 수준)의 `bitbucket-pipelines.yml`에 다음을 추가하거나 병합합니다:

```yaml
definitions:
  caches:
    nextcache: .next/cache
```

그런 다음 파이프라인 `step`의 `caches` 섹션에서 이를 참조하세요:

```yaml
- step:
    name: your_step_name
    caches:
      - node
      - nextcache
```

## Heroku
Heroku의 [사용자 정의 캐시](https://devcenter.heroku.com/articles/nodejs-support#custom-caching)를 사용하여 최상위 package.json에 `cacheDirectories` 배열을 추가합니다:

```json
"cacheDirectories": [".next/cache"]
```

## Azure Pipelines
Azure Pipelines의 [Cache task](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/cache)을 사용하여 `next build`를 실행하는 작업 이전 어딘가에 파이프라인 yaml 파일에 다음 작업을 추가합니다:

```yaml
- task: Cache@2
  displayName: 'Cache .next/cache'
  inputs:
    key: next | $(Agent.OS) | yarn.lock
    path: '$(System.DefaultWorkingDirectory)/.next/cache'
```

## Jenkins (Pipeline)
Jenkins의 [Job Cacher](https://www.jenkins.io/doc/pipeline/steps/jobcacher/) 플러그인을 사용하여 일반적으로 `next build` 또는 `npm install`를 실행하는 `Jenkinsfile`에 다음 빌드 단계를 추가합니다:

```yaml
stage("Restore npm packages") {
    steps {
        // Writes lock-file to cache based on the GIT_COMMIT hash
        writeFile file: "next-lock.cache", text: "$GIT_COMMIT"
 
        cache(caches: [
            arbitraryFileCache(
                path: "node_modules",
                includes: "**/*",
                cacheValidityDecidingFile: "package-lock.json"
            )
        ]) {
            sh "npm install"
        }
    }
}
stage("Build") {
    steps {
        // Writes lock-file to cache based on the GIT_COMMIT hash
        writeFile file: "next-lock.cache", text: "$GIT_COMMIT"
 
        cache(caches: [
            arbitraryFileCache(
                path: ".next/cache",
                includes: "**/*",
                cacheValidityDecidingFile: "next-lock.cache"
            )
        ]) {
            // aka `next build`
            sh "npm run build"
        }
    }
}
```