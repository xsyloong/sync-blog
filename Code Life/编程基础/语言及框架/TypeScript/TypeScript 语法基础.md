
## TypeScript 基础项目构建
### 初始化配置文件
```bash
# 初始化package.json
npm init -y

# 安装 TypeScript 和 Node.js 的类型定义文件作为开发依赖 
npm install typescript @types/node --save-dev 
# 初始化 tsconfig.json (TypeScript 编译配置文件) npx tsc --init

```

### 项目基础结构
```Plaintext
library-cli/
├── package.json
├── tsconfig.json
├── src/                      # 源码目录
│   ├── models/               # 数据模型 (类似 Java 的 Entity/DTO)
│   │   └── Book.ts
│   ├── services/             # 业务逻辑 (类似 Java 的 Service 层)
│   │   └── LibraryService.ts
│   └── index.ts              # 程序的入口 (类似 public static void main)
└── dist/                     # 编译产物目录 (运行 tsc 后自动生成)
```

### 配置编译选项

1. 打开生成的 `tsconfig.json` 
```json
{
  "compilerOptions": {
    "target": "ES2022",                // 编译目标版本
    "module": "CommonJS",              // 模块系统
    "rootDir": "./src",                // 源码根目录
    "outDir": "./dist",                // 编译后的输出目录
    "strict": true,                    // 开启严格类型检查
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  }
}
```
2. 编译与运行
```json
//运行npm run build 在build目录下就是编译好的js代码
//运行 npm run start 就会启动
{
  "name": "library-cli",
  "version": "1.0.0",
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "dev": "tsc --watch"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "typescript": "^5.0.0"
  }
}
```

## TypeScript 高级用法
