---
title: Go语言使用Mock单元测试
date: 2025-05-15 15:47:12
tags: [Go, Testing, Mock]
---

## 简介
使用Go开发Web项目时利用 **Mock** 工具进行单元测试。

### 安装工具
```bash
go install go.uber.org/mock/mockgen@latest
```

### 生成需要mock的实例
#### 方式1: 手敲命令行
```bash
# 为 service 生成 mock
mockgen -source=internal/service/user.go -package=svcmocks -destination=internal/service/mock/user.go
mockgen -source=internal/service/code.go -package=svcmocks -destination=internal/service/mock/code.go

# 为第三方包（以Redis为例）生成 mock
mockgen -package=redismocks -destination=internal/repository/cache/redismocks/cmdable.go github.com/redis/go-redis/v9 Cmdable
```

#### 方式2: 使用 `go:generate`
在项目根目录创建一个 `generate.go` 文件，集中管理所有 mock 生成指令。

```go
// 文件:generate.go
package main

//go:generate mockgen -source=internal/service/user.go -package=mocksvc -destination=mock/service/user.go
//go:generate mockgen -source=internal/service/code.go -package=mocksvc -destination=mock/service/code.go

//go:generate mockgen -source=internal/repository/user.go -package=mockrepo -destination=mock/repository/user.go
//go:generate mockgen -source=internal/repository/code.go -package=mockrepo -destination=mock/repository/code.go

//go:generate mockgen -source=internal/repository/cache/user.go -package=mockcache -destination=mock/repository/cache/user.go
//go:generate mockgen -source=internal/repository/cache/code.go -package=mockcache -destination=mock/repository/cache/code.go

//go:generate mockgen -source=internal/repository/dao/user.go -package=mockdao -destination=mock/repository/dao/user.go

// mock第三方依赖
//go:generate mockgen -package=mockredis -destination=mock/repository/cache/redis/cmdable.go github.com/redis/go-redis/v9 Cmdable
```
然后在项目目录下执行即可创建所有 mock 文件：
```bash
go generate ./...
```

### 进行测试
以测试 `UserHandler` 的 `SignUp` 方法为例。

```go
func TestUserHandler_SignUp(t *testing.T) {
	testCases := []struct {
		// 该test的名字
		name     string
		// NewUserHandler需要UserService和CodeService,这里返回mock的实例并进行EXPECT
		mock     func(ctrl *gomock.Controller) (service.UserService, service.CodeService)
		// 请求参数
		reqBody  string
		wantCode int
		wantBody string
	}{
		{
			name: "注册成功",
			mock: func(ctrl *gomock.Controller) (service.UserService, service.CodeService) {
				userSvc := svcmocks.NewMockUserService(ctrl)
				userSvc.EXPECT().SignUp(gomock.Any(), domain.User{
					Email:    "haha@qq.com",
					Password: "qwer#1234",
				}).Return(nil)
				codeSvc := svcmocks.NewMockCodeService(ctrl)
				return userSvc, codeSvc
			},
			reqBody: `{
				"email": "haha@qq.com",
				"password": "qwer#1234"
			}`,
			wantCode: http.StatusOK,
			wantBody: "注册成功",
		},
		{
			name: "邮箱格式不正确",
			mock: func(ctrl *gomock.Controller) (service.UserService, service.CodeService) {
				userSvc := svcmocks.NewMockUserService(ctrl)
				codeSvc := svcmocks.NewMockCodeService(ctrl)
				return userSvc, codeSvc
			},
			reqBody: `{
				"email": "hahaqq.com",
				"password": "qwer#1234"
			}`,
			wantCode: http.StatusBadRequest,
			wantBody: "邮箱格式不对",
		},
		{
			name: "参数错误,Bind失败",
			mock: func(ctrl *gomock.Controller) (service.UserService, service.CodeService) {
				userSvc := svcmocks.NewMockUserService(ctrl)
				codeSvc := svcmocks.NewMockCodeService(ctrl)
				return userSvc, codeSvc
			},
			reqBody: `{
				email": "hahaqq.com",
				"password": "qwer#1234"
			}`,
			wantCode: http.StatusBadRequest,
			wantBody: "参数错误",
		},
		{
			name: "邮箱冲突",
			mock: func(ctrl *gomock.Controller) (service.UserService, service.CodeService) {
				userSvc := svcmocks.NewMockUserService(ctrl)
				userSvc.EXPECT().SignUp(gomock.Any(), domain.User{
					Email:    "hahah@qq.com",
					Password: "qwer#1234",
				}).Return(service.ErrDuplicateEmail)
				codeSvc := svcmocks.NewMockCodeService(ctrl)
				return userSvc, codeSvc
			},
			reqBody: `{
				"email": "hahah@qq.com",
				"password": "qwer#1234"
			}`,
			wantCode: http.StatusInternalServerError,
			wantBody: "邮箱冲突",
		},
		{
			name: "系统错误",
			mock: func(ctrl *gomock.Controller) (service.UserService, service.CodeService) {
				userSvc := svcmocks.NewMockUserService(ctrl)
				userSvc.EXPECT().SignUp(gomock.Any(), domain.User{
					Email:    "hahah@qq.com",
					Password: "qwer#1234",
				}).Return(errors.New("随便一个异常"))
				codeSvc := svcmocks.NewMockCodeService(ctrl)
				return userSvc, codeSvc
			},
			reqBody: `{
				"email": "hahah@qq.com",
				"password": "qwer#1234"
			}`,
			wantCode: http.StatusInternalServerError,
			wantBody: "系统错误",
		},
		{
			name: "密码必须大于8位,包含数字、特殊字符",
			mock: func(ctrl *gomock.Controller) (service.UserService, service.CodeService) {
				userSvc := svcmocks.NewMockUserService(ctrl)
				codeSvc := svcmocks.NewMockCodeService(ctrl)
				return userSvc, codeSvc
			},
			reqBody: `{
				"email": "hahah@qq.com",
				"password": "qwer1234"
			}`,
			wantCode: http.StatusBadRequest,
			wantBody: "密码必须大于8位,包含数字、特殊字符",
		},
	}

	for _, tc := range testCases {
		t.Run(tc.name, func(t *testing.T) {
			// 固定写法
			ctrl := gomock.NewController(t)
			defer ctrl.Finish()

			// 启动服务器的流程
			server := gin.Default()
			userService, codeService := tc.mock(ctrl)
			handler := NewUserHandler(userService, codeService)
			handler.RegisterRouters(server)

			// 构建请求
			request, err := http.NewRequest(http.MethodPost, "/users/signup", bytes.NewReader([]byte(tc.reqBody)))
			request.Header.Set("Content-Type", "application/json")
			assert.NoError(t, err)
			
			// 将请求交给gin来接管,并将结果返回到recorder
			recorder := httptest.NewRecorder()
			server.ServeHTTP(recorder, request)
			
			// 进行校验
			assert.Equal(t, tc.wantCode, recorder.Code)
			assert.Equal(t, tc.wantBody, recorder.Body.String())
		})
	}
}
```

## 使用 `sqlmock` 来 mock 数据库(DB)
mock 数据库不使用 `gomock` 工具生成 mock 文件。

### 安装依赖
```bash
go get github.com/DATA-DOG/go-sqlmock
```
### 测试代码
```go
func TestUserDao_Insert(t *testing.T) {

	testcases := []struct {
		name string
		mock func(t *testing.T) *sql.DB

		user User

		wantErr error
	}{
		{
			name: "插入成功",
			mock: func(t *testing.T) *sql.DB {
				mockDb, mock, err := sqlmock.New()
				require.NoError(t, err)

				mock.ExpectExec("INSERT INTO .*").
					WillReturnResult(sqlmock.NewResult(1, 1))

				return mockDb
			},
			user: User{
				Nickname: "haha",
			},
			wantErr: nil,
		},
		{
			name: "邮箱冲突",
			mock: func(t *testing.T) *sql.DB {
				mockDb, mock, err := sqlmock.New()
				require.NoError(t, err)

				mock.ExpectExec("INSERT INTO .*").
					WillReturnError(&mysqlDriver.MySQLError{
						Number: 1062,
					})

				return mockDb
			},
			user: User{
				Email: sql.NullString{
					String: "haha@qq.com",
					Valid:  true,
				},
				Nickname: "haha",
			},
			wantErr: ErrDuplicateEmail,
		},
	}
	for _, tc := range testcases {
		t.Run(tc.name, func(t *testing.T) {
			mockDb := tc.mock(t)
			db, err := gorm.Open(mysql.New(mysql.Config{
				Conn:                      mockDb,
				SkipInitializeWithVersion: true,
			}), &gorm.Config{
				DisableAutomaticPing:   true,
				SkipDefaultTransaction: true})
			if err != nil {
				return
			}
			assert.NoError(t, err)

			dao := NewUserDao(db)
			err = dao.Insert(context.Background(), tc.user)
			assert.Equal(t, tc.wantErr, err)
		})
	}
}
```
