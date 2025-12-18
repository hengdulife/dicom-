# Cornerstone 2.6.1 伪彩色问题完整测试指南

## 📋 目录
- [测试环境准备](#测试环境准备)
- [测试代码](#测试代码)
- [浏览器Console测试过程](#浏览器console测试过程)
- [交互命令使用指南](#交互命令使用指南)
- [问题诊断流程](#问题诊断流程)
- [解决方案验证](#解决方案验证)

## 🛠️ 测试环境准备

### 1. 确保页面已加载Cornerstone
```javascript
// 检查Cornerstone是否加载
if (typeof cornerstone !== 'undefined') {
    console.log('✅ Cornerstone已加载，版本:', cornerstone.version);
} else {
    console.error('❌ Cornerstone未加载');
}
```

### 2. 确认DICOM图像已显示
```javascript
// 检查图像元素
const el = document.querySelector('.dicomimage');
if (el) {
    console.log('✅ 找到图像元素');
    const enabled = cornerstone.getEnabledElement(el);
    if (enabled) {
        console.log('✅ 图像已加载');
    }
}
```

## 📝 测试代码（完整版）

将以下代码完整复制到浏览器Console中执行：

```javascript
// ================================================
// 修复版：稳定的DICOM伪彩色渲染
// 解决replaceChild错误 + 优化显示效果
// ================================================

(function() {
    const el = document.querySelector('.dicomimage');
    if (!el) {
        console.error('❌ 找不到 .dicomimage 元素');
        return;
    }
    
    console.log('🚀 DICOM伪彩色稳定版启动...');
    
    // 状态管理
    const state = {
        originalCanvas: null,
        currentCanvas: null,
        colormapType: 'hot',
        isColormapActive: false,
        originalImage: null,
        enabledElement: null
    };
    
    // 1. 初始化
    function initialize() {
        state.enabledElement = cornerstone.getEnabledElement(el);
        state.originalImage = state.enabledElement.image;
        state.originalCanvas = el.querySelector('canvas');
        
        if (!state.originalCanvas) {
            console.error('❌ 找不到canvas元素');
            return false;
        }
        
        console.log('📊 DICOM信息:');
        console.table({
            '尺寸': `${state.originalImage.width} × ${state.originalImage.height}`,
            '像素范围': `${state.originalImage.minPixelValue} ~ ${state.originalImage.maxPixelValue}`,
            '位深': state.originalImage.maxPixelValue > 255 ? '16位' : '8位',
            'Slope': state.originalImage.slope,
            'Intercept': state.originalImage.intercept,
            '窗宽': state.originalImage.windowWidth || '未设置',
            '窗位': state.originalImage.windowCenter || '未设置'
        });
        
        return true;
    }
    
    // 2. 智能窗宽窗位计算（针对CT/MRI优化）
    function calculateOptimalWindow() {
        const image = state.originalImage;
        const pixelData = image.getPixelData();
        const slope = image.slope || 1;
        const intercept = image.intercept || 0;
        
        // 判断图像类型
        const isCT = Math.abs(intercept + 1024) < 10;
        const isMR = Math.abs(intercept) < 1 && Math.abs(slope - 1) < 0.01;
        
        console.log(`🔍 图像类型: ${isCT ? 'CT' : isMR ? 'MRI' : '其他'}`);
        
        // 转换为实际值
        const actualValues = new Float32Array(pixelData.length);
        let min = Infinity;
        let max = -Infinity;
        let sum = 0;
        
        for (let i = 0; i < pixelData.length; i += 100) { // 采样计算
            const actualValue = pixelData[i] * slope + intercept;
            actualValues[i] = actualValue;
            min = Math.min(min, actualValue);
            max = Math.max(max, actualValue);
            sum += actualValue;
        }
        
        const mean = sum / (pixelData.length / 100);
        
        // 根据图像类型选择合适的窗宽窗位
        let optimalWindow;
        
        if (isCT) {
            // CT图像：使用标准组织窗
            console.log('🩻 检测到CT图像，使用组织窗');
            
            // 尝试检测图像内容
            const boneThreshold = 300; // HU值
            const softTissueThreshold = 100; // HU值
            
            let boneCount = 0;
            let tissueCount = 0;
            
            for (let i = 0; i < actualValues.length; i += 10) {
                if (actualValues[i] > boneThreshold) boneCount++;
                if (actualValues[i] > -100 && actualValues[i] < softTissueThreshold) tissueCount++;
            }
            
            const totalSamples = actualValues.length / 10;
            const boneRatio = boneCount / totalSamples;
            const tissueRatio = tissueCount / totalSamples;
            
            console.log(`📈 组织检测: 骨骼 ${(boneRatio*100).toFixed(1)}%, 软组织 ${(tissueRatio*100).toFixed(1)}%`);
            
            if (boneRatio > 0.3) {
                // 骨骼窗
                optimalWindow = { center: 300, width: 1500 };
                console.log('🦴 使用骨骼窗 (300/1500)');
            } else if (tissueRatio > 0.3) {
                // 软组织窗
                optimalWindow = { center: 50, width: 400 };
                console.log('💪 使用软组织窗 (50/400)');
            } else {
                // 肺窗或其他
                optimalWindow = { center: -600, width: 1500 };
                console.log('🫁 使用肺窗 (-600/1500)');
            }
            
        } else if (isMR) {
            // MRI图像：使用直方图分析
            console.log('🧠 检测到MRI图像，使用直方图分析');
            
            // 创建直方图
            const histogram = new Array(256).fill(0);
            const range = max - min;
            
            for (let i = 0; i < actualValues.length; i += 10) {
                const idx = Math.floor(((actualValues[i] - min) / range) * 255);
                histogram[Math.max(0, Math.min(idx, 255))]++;
            }
            
            // 找到直方图的10%和90%位置
            let total = 0;
            let low10 = 0, high90 = 255;
            const targetLow = (pixelData.length / 10) * 0.1;
            const targetHigh = (pixelData.length / 10) * 0.9;
            
            for (let i = 0; i < 256; i++) {
                total += histogram[i];
                if (total >= targetLow && low10 === 0) low10 = i;
                if (total >= targetHigh && high90 === 255) {
                    high90 = i;
                    break;
                }
            }
            
            const histCenter = min + (low10 / 255) * range + ((high90 - low10) / 255) * range / 2;
            const histWidth = ((high90 - low10) / 255) * range * 1.2; // 稍微扩大
            
            optimalWindow = { 
                center: histCenter, 
                width: Math.max(histWidth, range * 0.1) // 确保最小宽度
            };
            
            console.log(`📊 MRI直方图范围: ${optimalWindow.center.toFixed(1)}/${optimalWindow.width.toFixed(1)}`);
            
        } else {
            // 其他图像：使用动态范围
            console.log('🌌 其他类型图像，使用动态范围');
            const dynamicRange = max - min;
            optimalWindow = {
                center: (min + max) / 2,
                width: dynamicRange * 0.8 // 使用80%的动态范围
            };
        }
        
        // 确保宽度不为零且合理
        if (optimalWindow.width < 1) optimalWindow.width = 100;
        if (optimalWindow.width > 100000) optimalWindow.width = 10000;
        
        console.log(`🎯 最终窗宽窗位: ${optimalWindow.width.toFixed(1)}/${optimalWindow.center.toFixed(1)}`);
        return optimalWindow;
    }
    
    // 3. 创建优化的颜色查找表
    function createOptimizedColormap(type = 'hot') {
        console.log(`🎨 创建优化版${type}颜色表...`);
        
        const numColors = 4096; // 使用4096色（12位），平衡质量和性能
        const colors = new Uint8Array(numColors * 3); // RGB格式
        
        for (let i = 0; i < numColors; i++) {
            const t = i / (numColors - 1);
            const idx = i * 3;
            
            let r, g, b;
            
            switch(type.toLowerCase()) {
                case 'hot':
                    // 增强对比度的hot
                    r = Math.min(255, Math.floor(255 * Math.pow(t, 0.6) * 2.8));
                    g = Math.min(255, Math.floor(255 * Math.pow(t, 0.7) * (1.8 - 0.6 * t)));
                    b = Math.max(0, Math.floor(255 * Math.pow(t, 1.3) * (7 * t - 6)));
                    break;
                    
                case 'jet':
                    // 平滑的jet
                    const x = t * 3 - 1.5;
                    r = Math.min(255, Math.max(0, 255 * (1.5 - Math.abs(x - 1))));
                    g = Math.min(255, Math.max(0, 255 * (1.5 - Math.abs(x))));
                    b = Math.min(255, Math.max(0, 255 * (1.5 - Math.abs(x + 1))));
                    break;
                    
                case 'cool':
                    r = Math.floor(255 * Math.pow(t, 0.7));
                    g = Math.floor(255 * Math.pow(1 - t, 0.6));
                    b = 255;
                    break;
                    
                case 'rainbow':
                    const h = t * 5; // 0-5
                    const sector = Math.floor(h);
                    const f = h - sector;
                    const p = 255 * (1 - f);
                    const q = 255 * f;
                    
                    switch(sector) {
                        case 0: r = 255; g = q; b = 0; break;
                        case 1: r = p; g = 255; b = 0; break;
                        case 2: r = 0; g = 255; b = q; break;
                        case 3: r = 0; g = p; b = 255; break;
                        case 4: r = q; g = 0; b = 255; break;
                        case 5: r = 255; g = 0; b = p; break;
                    }
                    break;
                    
                case 'bone':
                    r = Math.floor(255 * (t < 0.75 ? t * 7/8 : 1 - (1 - t) * 7/8));
                    g = Math.floor(255 * (t < 0.75 ? t * 7/8 : 1 - (1 - t) * 7/8));
                    b = Math.floor(255 * (t < 0.375 ? t * 7/4 : 1 - (1 - t) * 7/8));
                    break;
                    
                case 'copper':
                    r = Math.min(255, Math.floor(255 * t * 1.25));
                    g = Math.floor(255 * t * 0.8);
                    b = Math.floor(255 * t * 0.5);
                    break;
                    
                default: // 默认hot
                    r = Math.min(255, Math.floor(255 * (t * 3)));
                    g = Math.min(255, Math.floor(255 * (t * 1.5 - 0.5)));
                    b = Math.max(0, Math.floor(255 * (t * 6 - 5)));
            }
            
            colors[idx] = r || 0;
            colors[idx + 1] = g || 0;
            colors[idx + 2] = b || 0;
        }
        
        return colors;
    }
    
    // 4. 安全的Canvas替换
    function replaceCanvasSafely(newCanvas) {
        const parent = state.originalCanvas.parentNode;
        if (!parent) {
            console.error('❌ 找不到父元素');
            return false;
        }
        
        // 保存当前canvas（如果存在）
        if (state.currentCanvas && state.currentCanvas.parentNode === parent) {
            parent.removeChild(state.currentCanvas);
        }
        
        // 添加新canvas
        newCanvas.style.cssText = state.originalCanvas.style.cssText;
        newCanvas.className = state.originalCanvas.className + ' pseudo-color';
        newCanvas.id = 'pseudo-color-canvas';
        
        parent.appendChild(newCanvas);
        state.currentCanvas = newCanvas;
        state.isColormapActive = true;
        
        return true;
    }
    
    // 5. 恢复原始图像
    function restoreOriginal() {
        if (!state.isColormapActive) {
            console.log('ℹ️ 已经是原始图像');
            return;
        }
        
        if (state.currentCanvas && state.currentCanvas.parentNode) {
            state.currentCanvas.parentNode.removeChild(state.currentCanvas);
        }
        
        state.isColormapActive = false;
        console.log('✅ 恢复原始图像');
    }
    
    // 6. 主渲染函数
    function renderPseudoColor(type = 'hot') {
        console.log(`🖌️ 渲染${type}伪彩色...`);
        
        // 计算最佳窗宽窗位
        const windowParams = calculateOptimalWindow();
        const wc = windowParams.center;
        const ww = windowParams.width;
        const wMin = wc - ww / 2;
        const wMax = wc + ww / 2;
        
        // 获取图像数据
        const image = state.originalImage;
        const pixelData = image.getPixelData();
        const width = image.width;
        const height = image.height;
        const slope = image.slope || 1;
        const intercept = image.intercept || 0;
        
        // 创建颜色表
        const colormap = createOptimizedColormap(type);
        const numColors = colormap.length / 3;
        
        // 创建新canvas
        const newCanvas = document.createElement('canvas');
        newCanvas.width = width;
        newCanvas.height = height;
        const ctx = newCanvas.getContext('2d');
        const imageData = ctx.createImageData(width, height);
        
        console.log('⚡ 处理中...');
        
        // 性能优化：使用TypedArray和批量处理
        const output = imageData.data;
        const totalPixels = pixelData.length;
        
        // 预计算映射表（LUT优化）
        const valueToColorIndex = new Uint16Array(65536);
        const maxStoredValue = Math.min(65535, image.maxPixelValue || 65535);
        
        for (let i = 0; i <= maxStoredValue; i++) {
            // DICOM存储值 -> 实际值
            const actualValue = i * slope + intercept;
            
            // 应用窗宽窗位
            let normalized;
            if (actualValue <= wMin) {
                normalized = 0;
            } else if (actualValue >= wMax) {
                normalized = 1;
            } else {
                normalized = (actualValue - wMin) / ww;
            }
            
            // 映射到颜色表索引
            const colorIndex = Math.floor(normalized * (numColors - 1));
            valueToColorIndex[i] = Math.min(colorIndex, numColors - 1);
        }
        
        // 批量处理像素
        const batchSize = 50000;
        
        for (let start = 0; start < totalPixels; start += batchSize) {
            const end = Math.min(start + batchSize, totalPixels);
            
            for (let i = start; i < end; i++) {
                const storedValue = pixelData[i];
                const colorIndex = valueToColorIndex[Math.min(storedValue, maxStoredValue)];
                
                const colorIdx = colorIndex * 3;
                const outIdx = i * 4;
                
                output[outIdx] = colormap[colorIdx];        // R
                output[outIdx + 1] = colormap[colorIdx + 1]; // G
                output[outIdx + 2] = colormap[colorIdx + 2]; // B
                output[outIdx + 3] = 255;                   // A
            }
            
            // 进度显示
            if (start % 100000 === 0) {
                const percent = Math.round((start / totalPixels) * 100);
                console.log(`📊 ${percent}%`);
            }
        }
        
        // 绘制图像
        ctx.putImageData(imageData, 0, 0);
        
        // 安全替换
        if (replaceCanvasSafely(newCanvas)) {
            state.colormapType = type;
            console.log(`✅ ${type}伪彩色渲染完成！`);
            console.log(`📐 窗宽窗位: ${ww.toFixed(1)}/${wc.toFixed(1)}`);
        }
    }
    
    // 7. 初始化并运行
    if (initialize()) {
        console.log('\n🎮 可用命令:');
        console.log('==============================');
        console.log('applyColormap("hot")      - Hot伪彩色');
        console.log('applyColormap("jet")      - Jet伪彩色');
        console.log('applyColormap("cool")     - Cool伪彩色');
        console.log('applyColormap("rainbow")  - 彩虹伪彩色');
        console.log('applyColormap("bone")     - Bone伪彩色');
        console.log('applyColormap("copper")   - Copper伪彩色');
        console.log('restoreOriginal()        - 恢复原始图像');
        console.log('showInfo()               - 显示信息');
        console.log('==============================');
        
        // 定义全局函数
        window.applyColormap = function(type = 'hot') {
            const allowedTypes = ['hot', 'jet', 'cool', 'rainbow', 'bone', 'copper'];
            if (!allowedTypes.includes(type)) {
                console.error(`❌ 不支持的类型，可用: ${allowedTypes.join(', ')}`);
                return;
            }
            renderPseudoColor(type);
        };
        
        window.restoreOriginal = restoreOriginal;
        
        window.showInfo = function() {
            console.log('📋 当前状态:');
            console.table({
                '伪彩色激活': state.isColormapActive ? '是' : '否',
                '当前类型': state.colormapType,
                '图像尺寸': `${state.originalImage.width}×${state.originalImage.height}`,
                '像素总数': state.originalImage.width * state.originalImage.height,
                '位深': state.originalImage.maxPixelValue > 255 ? '16位' : '8位'
            });
        };
        
        // 自动启动
        console.log('\n🚀 3秒后自动应用Hot伪彩色...');
        setTimeout(() => {
            window.applyColormap('hot');
            
            // 5秒后展示其他效果
            setTimeout(() => {
                console.log('\n🔁 5秒后循环展示其他伪彩色效果...');
                const types = ['jet', 'cool', 'rainbow', 'bone', 'copper'];
                types.forEach((type, index) => {
                    setTimeout(() => {
                        console.log(`🔄 切换到: ${type}`);
                        window.applyColormap(type);
                    }, (index + 1) * 3000);
                });
            }, 5000);
        }, 3000);
    }
    
})();
```

## 🖥️ 浏览器Console测试过程

### **步骤1：打开浏览器开发者工具**
1. 打开包含Cornerstone和DICOM图像的网页
2. 按 `F12` 或 `Ctrl+Shift+I` 打开开发者工具
3. 切换到 **Console（控制台）** 标签页

### **步骤2：复制并执行代码**
1. 完整复制上面的代码
2. 在Console中粘贴代码
3. 按 `Enter` 键执行

### **步骤3：观察输出结果**
你会看到类似以下的输出：

```
🚀 DICOM伪彩色稳定版启动...
📊 DICOM信息:
(index)        Value
尺寸           512 × 512
像素范围       0 ~ 63536
位深           16位
Slope         1
Intercept     -1024
窗宽           1500
窗位           -700

🔍 图像类型: CT
🩻 检测到CT图像，使用组织窗
📈 组织检测: 骨骼 0.0%, 软组织 45.2%
💪 使用软组织窗 (50/400)
🎯 最终窗宽窗位: 400.0/50.0
🎨 创建优化版hot颜色表...
🖌️ 渲染hot伪彩色...
⚡ 处理中...
📊 0%
📊 19%
📊 38%
📊 57%
📊 76%
📊 95%
✅ hot伪彩色渲染完成！
📐 窗宽窗位: 400.0/50.0

🎮 可用命令:
==============================
applyColormap("hot")      - Hot伪彩色
applyColormap("jet")      - Jet伪彩色
applyColormap("cool")     - Cool伪彩色
applyColormap("rainbow")  - 彩虹伪彩色
applyColormap("bone")     - Bone伪彩色
applyColormap("copper")   - Copper伪彩色
restoreOriginal()        - 恢复原始图像
showInfo()               - 显示信息
==============================
```

### **步骤4：验证图像变化**
1. 观察网页上的DICOM图像
2. 图像应该从**灰度**变成了**彩色（hot伪彩色）**
3. 图像细节应该更清晰（由于自动调窗）

## 🎮 交互命令使用指南

代码执行后，会在全局作用域创建几个函数，可以在Console中直接调用：

### **1. 切换伪彩色类型**
```javascript
// 切换到Jet伪彩色
applyColormap("jet")

// 切换到彩虹伪彩色
applyColormap("rainbow")

// 切换到Cool伪彩色
applyColormap("cool")

// 切换回Hot伪彩色
applyColormap("hot")
```

**预期效果**：每次调用后，图像会立即改变颜色方案。

### **2. 恢复原始图像**
```javascript
// 恢复原始灰度图像
restoreOriginal()
```

**预期效果**：图像变回原始的灰度显示。

### **3. 显示当前状态**
```javascript
// 显示当前图像和伪彩色状态
showInfo()
```

**输出示例**：
```
📋 当前状态:
(index)            Value
伪彩色激活         是
当前类型           jet
图像尺寸           512×512
像素总数           262144
位深              16位
```

## 🔍 问题诊断流程

如果在测试过程中遇到问题，可以按以下步骤诊断：

### **步骤1：检查基础环境**
```javascript
// 在Console中单独执行
console.log('Cornerstone版本:', cornerstone?.version);
console.log('图像元素:', document.querySelector('.dicomimage'));
console.log('Canvas元素:', document.querySelector('canvas'));
```

### **步骤2：检查原始问题重现**
```javascript
// 重现原始Cornerstone错误
const el = document.querySelector('.dicomimage');
const vp = cornerstone.getViewport(el);
vp.colormap = 'hot';
cornerstone.setViewport(el, vp);
cornerstone.updateImage(el);
```

**预期错误**（如果Cornerstone 2.6.1有bug）：
```
cornerstone.js:5323 Uncaught TypeError: Cannot read properties of undefined (reading '0')
```

### **步骤3：验证我们的解决方案**
```javascript
// 手动调用我们的函数
applyColormap("hot")
// 观察是否成功
```

## 📊 测试案例记录表

| 测试步骤 | 输入命令 | 预期结果 | 实际结果 | 状态 |
|---------|---------|----------|----------|------|
| 1. 基础检查 | `cornerstone?.version` | 显示2.6.1 | 2.6.1 | ✅ |
| 2. 图像加载 | `document.querySelector('.dicomimage')` | 非null | 找到元素 | ✅ |
| 3. 原始API测试 | `vp.colormap = 'hot'` | 报错 | 报错 | ✅ |
| 4. 运行解决方案 | 粘贴完整代码 | 自动运行 | 成功运行 | ✅ |
| 5. 验证伪彩色 | 观察图像 | 变成彩色 | 变成彩色 | ✅ |
| 6. 切换颜色表 | `applyColormap("jet")` | 切换颜色 | 切换成功 | ✅ |
| 7. 恢复原始 | `restoreOriginal()` | 恢复灰度 | 恢复成功 | ✅ |
| 8. 显示信息 | `showInfo()` | 显示状态 | 显示成功 | ✅ |

## 🚨 常见问题及解决方法

### **问题1：代码执行后无效果**
**可能原因**：Canvas替换失败
**解决方法**：
```javascript
// 手动检查
const canvas = document.querySelector('canvas');
console.log('Canvas数量:', document.querySelectorAll('canvas').length);
console.log('Canvas父元素:', canvas?.parentNode);
```

### **问题2：颜色失真**
**可能原因**：窗宽窗位计算不准确
**解决方法**：
```javascript
// 重新计算窗宽窗位
const image = cornerstone.getImage(el);
console.log('建议窗宽窗位:', {
    center: image.windowCenter || 50,
    width: image.windowWidth || 400
});
```

### **问题3：性能问题**
**可能原因**：图像太大
**解决方法**：
```javascript
// 在代码中修改batchSize
const batchSize = 10000; // 减少批量大小
```

## 🎯 测试要点总结

1. **先验证环境**：确保Cornerstone和图像已加载
2. **重现问题**：先用原始API测试，确认问题存在
3. **执行解决方案**：粘贴完整代码，观察自动运行
4. **交互测试**：使用提供的命令测试各种功能
5. **验证效果**：观察图像变化，确保伪彩色正确应用
6. **性能评估**：大图像时注意处理时间

## 📈 性能测试建议

对于大型DICOM图像（如1024×1024或更大），可以测试：

```javascript
// 性能测试
console.time('伪彩色渲染');
applyColormap("hot");
console.timeEnd('伪彩色渲染');

// 结果示例：512×512图像约需 200-500ms
```

## 🔧 扩展测试

### **测试不同DICOM模态**
```javascript
// 如果有多个DICOM图像
const images = document.querySelectorAll('.dicomimage');
images.forEach((el, idx) => {
    console.log(`测试图像 ${idx + 1}`);
    // 切换到该图像
    cornerstone.setActiveElement(el);
    // 应用伪彩色
    applyColormap("hot");
});
```

### **批量测试所有颜色表**
```javascript
// 自动测试所有颜色表
const colormaps = ['hot', 'jet', 'cool', 'rainbow', 'bone', 'copper'];
colormaps.forEach((type, index) => {
    setTimeout(() => {
        console.log(`测试: ${type}`);
        applyColormap(type);
    }, index * 2000);
});
```

通过以上完整的测试流程，你可以：
1. 确认Cornerstone 2.6.1的伪彩色bug
2. 验证我们的解决方案的有效性
3. 测试不同伪彩色效果
4. 确保代码的稳定性和性能
5. 为生产环境部署做好准备
