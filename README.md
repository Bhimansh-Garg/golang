name: Workflow-Pipeline
on:
  push:
    branches:
      - main
      - development
    paths-ignore:
      - 'docs/**'
  pull_request:
    branches:
      - main
      - development
    paths-ignore:
      - 'docs/**'
jobs:
  Example-Unit-Testing:
    name: Example Unit Testing (v${{ matrix.go-version }})🛠
    runs-on: ubuntu-latest
    strategy:
      matrix:
        go-version: ['1.22', '1.21']
    services:
      kafka:
        image: bitnami/kafka:3.4
        ports:
          - "9092:9092"
        env:
          KAFKA_ENABLE_KRAFT: yes
          KAFKA_CFG_PROCESS_ROLES: broker,controller
          KAFKA_CFG_CONTROLLER_LISTENER_NAMES: CONTROLLER
          KAFKA_CFG_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
          KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
          KAFKA_CFG_ADVERTISED_LISTENERS: PLAINTEXT://127.0.0.1:9092
          KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE: true
          KAFKA_BROKER_ID: 1
          KAFKA_CFG_CONTROLLER_QUORUM_VOTERS: 1@127.0.0.1:9093
          ALLOW_PLAINTEXT_LISTENER: yes
          KAFKA_CFG_NODE_ID: 1
      redis:
        image: redis:7.0.5
        ports:
          - "2002:6379"
        options: "--entrypoint redis-server"
      mysql:
        image: mysql:8.2.0
        ports:
          - "2001:3306"
        env:
          MYSQL_ROOT_PASSWORD: "password"
          MYSQL_DATABASE: "test"
    steps:
      - name: Checkout code into go module directory
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Set up Go ${{ matrix.go-version }}
        uses: actions/setup-go@v5
        with:
          go-version: ${{ matrix.go-version }}
        id: Go
      - name: Get dependencies
        run: |
          go mod download
      - name: Start Zipkin
        run: docker run -d -p 2005:9411 openzipkin/zipkin:latest
      - name: Test
        run: |
          export APP_ENV=test
          go test gofr.dev/examples/... -v -short -coverprofile packageWithpbgo.cov -coverpkg=gofr.dev/examples/...
          grep -vE '^gofr\.dev\/.*\.pb\.go' packageWithpbgo.cov > profile.cov
          go tool cover -func profile.cov
      - name: Upload Test Coverage
        if: ${{ matrix.go-version == '1.22'}}
        uses: actions/upload-artifact@v4
        with:
          name: Example-Test-Report
          path: profile.cov
  PKG-Unit-Testing:
    name: PKG Unit Testing (v${{ matrix.go-version }})🛠
    runs-on: ubuntu-latest
    strategy:
      matrix:
        go-version: ['1.22', '1.21']
    steps:
      - name: Checkout code into go module directory
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Set up Go ${{ matrix.go-version }}
        uses: actions/setup-go@v5
        with:
          go-version: ${{ matrix.go-version }}
        id: Go
      - name: Get dependencies
        run: |
          go mod download
      - name: Test
        run: |
          export APP_ENV=test
          go test gofr.dev/pkg/... -v -short -coverprofile package.cov -coverpkg=gofr.dev/pkg/...
          grep -v '/mock_' package.cov > profile.cov
          go tool cover -func profile.cov
      - name: Upload Test Coverage
        if: ${{ matrix.go-version == '1.22'}}
        uses: actions/upload-artifact@v4
        with:
          name: PKG-Coverage-Report
          path: profile.cov
  parse_coverage:
    name: Code Coverage
    runs-on: ubuntu-latest
    needs: [ Example-Unit-Testing,PKG-Unit-Testing]
    steps:
      - name: Check out code into the Go module directory
        uses: actions/checkout@v4
      - name: Download Coverage Report
        uses: actions/download-artifact@v4
        with:
          path: artifacts
      - name: Merge Coverage Files
        working-directory: artifacts
        run: |
          awk '!/^mode: / && FNR==1{print "mode: set"} {print}' ./Example-Test-Report/profile.cov > merged_profile.cov
          tail -n +2 ./PKG-Coverage-Report/profile.cov >> merged_profile.cov
      - name: Parse code-coverage value
        working-directory: artifacts
        run: |
          codeCoverage=$(go tool cover -func=merged_profile.cov | grep total | awk '{print $3}')
          codeCoverage=${codeCoverage%?}
          echo "CODE_COVERAGE=$codeCoverage" >> $GITHUB_ENV
#      - name: Check if code-coverage is greater than threshold
#        run: |
#          codeCoverage=${{ env.CODE_COVERAGE }}
#          codeCoverage=${codeCoverage%??}
#          if [[ $codeCoverage -lt 92 ]]; then echo "code coverage cannot be less than 92%, currently its ${{ env.CODE_COVERAGE }}%" && exit 1; fi;
  upload_coverage:
    name: Upload Coverage📊
    runs-on: ubuntu-latest
    needs: [Example-Unit-Testing,PKG-Unit-Testing]
    if: ${{ github.event_name == 'push' && github.ref == 'refs/heads/development'}}
    steps:
      - name: Check out code into the Go module directory
        uses: actions/checkout@v4
      - name: Download Coverage Report
        uses: actions/download-artifact@v4
        with:
          path: artifacts
      - name: Merge Coverage Files
        working-directory: artifacts
        run: |
          awk '!/^mode: / && FNR==1{print "mode: set"} {print}' ./Example-Test-Report/profile.cov > merged_profile.cov
          tail -n +2 ./PKG-Coverage-Report/profile.cov >> merged_profile.cov
      - name: Upload
        uses: paambaati/codeclimate-action@v8.0.0
        uses: paambaati/codeclimate-action@v9.0.0
        env:
          CC_TEST_REPORTER_ID: ${{ secrets.CC_TEST_REPORTER_ID }}
        with:
