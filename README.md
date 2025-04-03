# Vue 3 SVG Icon 组件文档

## 概述

此组件用于在 Vue 3 项目中显示 SVG 图标，支持通过 `el-tooltip` 添加悬浮提示信息，支持动态加载和显示不同的 SVG 图标。该组件能够自定义图标的尺寸、颜色、位置等样式，并通过插件系统动态加载 SVG 文件。

## 安装依赖

1. 安装 `pnpm`（如果你没有安装）：
   ```bash
   npm install -g pnpm
   ```

2. 安装组件所需的依赖：
   ```bash
   pnpm install
   ```

3. 安装并配置 `SvgLoader` 插件用于自动加载 SVG 文件：
   ```javascript
   import { SvgLoader } from './svg-loader';
   
   SvgLoader('./src/assets/svg/');  // 配置你的 SVG 图标文件路径
   ```

## 组件使用

### 1. 引入组件

在你的 Vue 3 项目中引入该 SVG 组件并使用它：

```vue
<template>
  <div>
    <SvgIcon
      :icon="'home'"
      :size="'24px'"
      :color="'#000'"
      :tooltip="'Go to Home'"
      :placement="'bottom'"
    />
  </div>
</template>

<script setup lang="ts">
import SvgIcon from '@/components/SvgIcon.vue';
</script>
```

### 2. 组件属性

| 属性       | 类型            | 默认值       | 说明                                                  |
|------------|-----------------|--------------|-------------------------------------------------------|
| `icon`     | `string`        | 无           | 图标名称，需与 SVG 文件名对应。                        |
| `size`     | `string | number`| `1em`        | 图标大小，支持 `px`, `em`, `%` 等单位。               |
| `color`    | `string`        | `currentColor`| 图标颜色，支持任何有效的 CSS 颜色值。                 |
| `width`    | `string | number`| 无           | 图标宽度，可选。                                      |
| `height`   | `string | number`| 无           | 图标高度，可选。                                      |
| `tooltip`  | `string`        | 无           | 悬浮提示内容，若有该属性，则显示 Tooltip。             |
| `placement`| `string`        | `top`        | Tooltip 显示的位置，可选 `top`, `bottom`, `left`, `right`。 |

### 3. 图标样式

通过 `size`, `width`, `height` 和 `color` 属性，你可以灵活地定制图标的样式：

```vue
<SvgIcon
  :icon="'settings'"
  :size="'48px'"
  :color="'#FF5733'"
  :tooltip="'Settings Icon'"
  :placement="'top'"
/>
```

### 4. 显示 Tooltip

如果你想为图标添加悬浮提示文本，可以通过 `tooltip` 属性传入提示内容。

```vue
<SvgIcon
  :icon="'search'"
  :size="'32px'"
  :tooltip="'Search icon'"
/>
```

### 5. 动态加载 SVG 图标

通过 `SvgLoader` 插件，你可以动态加载 SVG 文件并在 Vue 组件中使用。确保你已经将图标文件放入指定的目录（如 `./src/assets/svg/`），并通过 `SvgLoader` 插件进行处理。

### 6. 示例

以下是一个完整的示例，展示如何在 Vue 组件中使用 SVG 图标，并通过 `SvgLoader` 动态加载 SVG 文件：

```vue
<template>
  <div>
    <SvgIcon
      :icon="'home'"
      :size="'24px'"
      :color="'#333'"
      :tooltip="'Home Icon'"
      :placement="'right'"
    />
  </div>
</template>

<script setup lang="ts">
import { SvgIcon } from '@/components/SvgIcon.vue';
import { SvgLoader } from './svg-loader';

// 通过插件动态加载 SVG 文件
SvgLoader('./src/assets/svg/');
</script>
```

### 7. 插件配置

以下是 `SvgLoader` 插件的代码，用于自动加载 SVG 图标并将其嵌入 HTML 中：

```javascript
import { readFileSync, readdirSync } from 'fs'
import path from 'path'
import crypto from 'crypto'

let idPerfix = ''

const svgTitle = /<svg([^>+].*?)>/
const clearHeightWidth = /(width|height)="([^>+].*?)"/g
const hasViewBox = /(viewBox="[^>+].*?")/g
const clearReturn = /(\r)|(\n)/g

function generateUniqueGradientId(fileName: string, baseId: string) {
  const hash = crypto.createHash('md5').update(fileName).digest('hex').substring(0, 6)
  return `${baseId}-${hash}`
}

function findSvgFile(dir: string) {
  const svgRes: any = []
  const dirents = readdirSync(dir, { withFileTypes: true })

  for (const dirent of dirents) {
    if (dirent.isDirectory()) {
      svgRes.push(...findSvgFile(path.join(dir, dirent.name)))
    } else {
      try {
        let svgContent = readFileSync(path.join(dir, dirent.name), 'utf-8')
          .replace(clearReturn, '')
          .replace(svgTitle, ($1, $2) => {
            let width = 0
            let height = 0
            let content = $2.replace(clearHeightWidth, (s1: any, s2: any, s3: any) => {
              if (s2 === 'width') {
                width = s3
              } else if (s2 === 'height') {
                height = s3
              }
              return ''
            })
            if (!hasViewBox.test($2)) {
              content += `viewBox="0 0 ${width} ${height}"`
            }
            return `<symbol id="${idPerfix}-${dirent.name.replace('.svg', '')}" ${content}>`
          })
          .replace('</svg>', '</symbol>')

        const gradientMap: Record<string, string> = {}

        svgContent = svgContent.replace(
          /(<(linearGradient|radialGradient)[^>]*id=")([^"]*)(")/g,
          (match, p1, p2, gradientId, p4) => {
            const uniqueId = generateUniqueGradientId(dirent.name, gradientId)
            gradientMap[gradientId] = uniqueId
            return `${p1}${uniqueId}${p4}`
          }
        )

        svgContent = svgContent.replace(/url\(#([^)]+)\)/g, (match, oldId) => {
          const newId = gradientMap[oldId]
          return newId ? `url(#${newId})` : match
        })

        svgContent = svgContent.replace(/(xlink:href|href)="#([^"]+)"/g, (match, attr, oldId) => {
          const newId = gradientMap[oldId]
          return newId ? `${attr}="#${newId}"` : match
        })

        svgRes.push(svgContent)
      } catch (error) {
        console.log('findSvgFile ~ error:', error)
      }
    }
  }
  return svgRes
}

export const SvgLoader = (path: any, perfix = 'icon') => {
  if (path === '') return
  idPerfix = perfix
  const res = findSvgFile(path)
  return {
    name: 'svg-transform',
    transformIndexHtml(html: any) {
      return html.replace(
        '<body>',
        `
          <body>
            <svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" style="position: absolute; width: 0; height: 0">
              ${res.join('')}
            </svg>
        `
      )
    }
  }
}
```

## 结论

通过该组件和插件，你可以轻松地在 Vue 3 项目中使用 SVG 图标，并通过简洁的配置来管理图标的大小、颜色和样式。插件支持自动加载和管理 SVG 图标，使得图标的使用更加灵活和方便。
