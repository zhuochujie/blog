---
title: Go语言使用Mock单元测试
date: 2025-05-15 15:47:12
tags:
---

## 简介
使用Go开发Web项目时利用Mock工具进行单元测试

### 安装工具
```bash
go install go.uber.org/mock/mockgen@latest
```

### 生成需要mock的实例
```
mockgen -source=internal/service/user.go -package=svcmocks -destination=internal/service/mock/user.go
mockgen -source=internal/service/code.go -package=svcmocks -destination=internal/service/mock/code.go
```

### 进行测试
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