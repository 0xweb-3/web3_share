# 01.开发环境的构建
## 安装node.js
按照[官网](https://nodejs.org/zh-cn)指引下载对应的`node.js`版本

## 构建typescript
```shell
# 创建项目目录
mkdir ts_eth_hd_wallet
cd ts_eth_hd_wallet
npm init -y

# 安装ts-node和typescript
npm install -g ts-node typescript
npm install --save-dev webpack webpack-cli ts-loader
npx tsc --init # 创建ts配置文件
```

## 测试运行环境
```shell
mkdir src
cd src
touch index.ts
```
`src/index.ts`下内容为
```typescript
const message: string = 'Hello, TypeScript!';
console.log(message);
```
运行 ts-node src/index.ts 后，输出将显示在命令行中：
```shell
Hello, TypeScript!
```
## 构建测试环境
安装
```shell
npm install ts-jest jest @types/jest --save-dev
```
修改配置文件`tsconfig.json`确保配置【一般不用修改】：
```shell
{
  "compilerOptions": {
    "target": "ES6",
    "module": "commonjs",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  }
}
```
配置`Jest`,在项目根目录下创建一个`jest.config.js`文件，并添加以下内容：
```javascript
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
};
```
修改`src/index.ts`
```typescript
// src/index.ts
export const sum = (a: number, b: number): number => {
  return a + b;
};
```
创建测试文件`index.test.ts`
```typescript
// src/index.test.ts
import { sum } from './index';

test('adds 1 + 2 to equal 3', () => {
  expect(sum(1, 2)).toBe(3);
});
```
修改`package.json`:
```javascript
{
  "scripts":{
    "test": "jest"
  }
}
```
运行
```shell
npm test
```
