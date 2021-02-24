# Self-hosted Runner

GitHub Actions Self-hosted Runner running on AWS CodeBuild

<img src="https://github.com/rvillage/self_hosted_runner/blob/master/images/system_configuration.png?raw=true" alt="System Configuration" width="800"/>

## インストール済

- awscli
- docker
- github-cli

## 動作確認済み

- actions/checkout@v2
- actions/github-script@v2
- aws-actions/configure-aws-credentials@v1
- ruby/setup-ruby@v1

## Setup

### AWS

1. `.deploy/cloud_formation/codebuild.yml`でCloudFormationスタックの作成
2. docker imageのビルドしてECRにプッシュ
```sh
DOCKER_BUILDKIT=1 docker build -t xxx.dkr.ecr.ap-northeast-1.amazonaws.com/self_hosted_runner:latest -f Dockerfile .
docker push xxx.dkr.ecr.ap-northeast-1.amazonaws.com/self_hosted_runner:latest
```
3. IAMユーザ`github-actions-user`のアクセスキーをGitHubリポジトリに設定
4. RunnerToken発行用のPersonal access tokenをGitHubリポジトリに設定
5. エラー対策でオフラインRunnerを追加しておく
  - https://github.com/xxx/xxx/settings/actions/add-new-runner にアクセスしてURLとTOKENをメモ
```sh
docker run -it --rm local/runner ./actions-runner/config.sh --unattended --name do_not_remove --url URL --token TOKEN
```
6. `.github/workflows/test.yml`の作成
```yml
name: SelfHostedRunner Test
on: push

jobs:
  setup:
    runs-on: ubuntu-20.04
    steps:
      - uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-northeast-1
      - name: Start SelfHostedRunner
        env:
          api_endpoint: https://api.github.com/repos/${{ github.repository }}/actions/runners/registration-token
        run: |
          runner_token=$(curl -sX POST -H "Authorization: token ${{ secrets.PERSONAL_ACCESS_TOKEN }}" $api_endpoint | jq -r '.token')
          aws codebuild start-build \
            --project-name SelfHostedRunner \
            --compute-type-override BUILD_GENERAL1_SMALL \
            --environment-variables-override \
                name=REPOSITORY_NAME,value=${{ github.repository }},type=PLAINTEXT \
                name=RUNNER_TOKEN,value=$runner_token,type=PLAINTEXT
  first_job:
    needs: setup
    runs-on: self-hosted
    steps:
      - run: echo "Hello SelfHostedRunner!"
```

### Local

1. (初回のみ) docker imageのビルド
```sh
DOCKER_BUILDKIT=1 docker build -t local/runner -f Dockerfile .
docker-compose up --no-start
```
2. (初回のみ) `docker-compose.yml.sample`から`docker-compose.yml`にコピーしてenvを設定
3. 起動
```sh
docker-compose start runner && docker-compose logs -f runner
```
4. 停止
```sh
docker-compose stop runner
```
5. クリーンアップ
```sh
docker-compose down --volumes
```
