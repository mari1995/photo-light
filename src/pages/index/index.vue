<template>
  <view class="wrapper">
    <view 
      class="container" 
      :style="{ backgroundColor: currentColor }"
      @tap="openPanel"
    ></view>

    <view 
      v-if="showPanel" 
      class="mask" 
      @tap="closePanel"
    ></view>

    <view 
      class="color-panel" 
      v-if="showPanel" 
      @tap.stop
      @touchmove.stop.prevent
    >
      <view 
        class="favorite-btn glass-effect"
        @tap.stop="toggleFavorite"
      >
        <text class="iconfont" :class="{ 'icon-active': isFavorite }">★</text>
      </view>

      <view class="panel-header glass-effect">
        <view class="tab-group">
          <view 
            class="tab-item glass-effect" 
            :class="{ active: mode === 'preset' }"
            @tap="mode = 'preset'"
          >预设色</view>
          <view 
            class="tab-item glass-effect" 
            :class="{ active: mode === 'wheel' }"
            @tap="mode = 'wheel'"
          >色轮</view>
          <view 
            class="tab-item glass-effect" 
            :class="{ active: mode === 'favorite' }"
            @tap="mode = 'favorite'"
          >收藏</view>
        </view>
      </view>

      <view v-if="mode === 'preset'" class="preset-mode" backgroundColor="rgba(255, 255, 255, 0.2)">
        <scroll-view class="color-grid" scroll-y>
          <view class="color-grid-content">
            <view 
              class="grid-item" 
              v-for="(color, index) in presetColors" 
              :key="index"
              :class="{ active: currentColorIndex === index }"
              @tap="selectColor(index)"
            >
              <view 
                class="color-preview" 
                :style="{ backgroundColor: color.value }"
              ></view>
              <text class="color-name">{{ color.name }}</text>
            </view>
          </view>
        </scroll-view>
      </view>

      <view v-else-if="mode === 'wheel'" class="wheel-mode">
        <view class="wheel-container">
          <canvas 
            id="colorWheel"
            canvas-id="colorWheel" 
            class="wheel-canvas"
            @touchstart="handleWheelTouch"
            @touchmove="handleWheelTouch"
          ></canvas>
          <view 
            class="color-picker"
            :style="{
              left: `${pointerPosition.x}px`,
              top: `${pointerPosition.y}px`,
              backgroundColor: wheelColor,
              borderColor: '#fff'
            }"
          ></view>
        </view>
      </view>

      <view v-else-if="mode === 'favorite'" class="favorite-mode">
        <scroll-view class="color-grid" scroll-y>
          <view class="color-grid-content">
            <view 
              class="grid-item" 
              v-for="(color, index) in favoriteColors" 
              :key="index"
              :class="{ active: currentColor === color.value }"
            >
              <view 
                class="color-preview" 
                :style="{ backgroundColor: color.value }"
                @tap="selectFavoriteColor(color)"
              ></view>
              <text class="color-name">{{ color.name }}</text>
              <view 
                class="delete-btn glass-effect"
                @tap.stop="deleteFavoriteColor(index)"
              >
                <text class="delete-icon">×</text>
              </view>
            </view>
          </view>
        </scroll-view>
      </view>

      <view class="control-panel glass-effect">
        <view class="control-item">
          <text class="control-label">亮度</text>
          <slider
            class="control-slider"
            :value="brightness"
            min="0"
            max="100"
            @change="handleBrightnessChange"
            show-value
            block-size="16"
            block-color="#FE93B3"
            activeColor="#FFD6E4"
            backgroundColor="rgba(255, 255, 255, 0.2)"
          />
        </view>
        <view class="control-item">
          <text class="control-label">深浅</text>
          <slider
            class="control-slider"
            :value="colorValue"
            min="0"
            max="100"
            @change="handleColorValueChange"
            show-value
            block-size="16"
            block-color="#FE93B3"
            activeColor="#FFD6E4"
            backgroundColor="rgba(255, 255, 255, 0.2)"
          />
        </view>
      </view>
    </view>
  </view>
</template>

<script setup>
import { ref, computed, onMounted, watch, nextTick } from 'vue'
import tinycolor from 'tinycolor2'

// 选择模式
const mode = ref('preset')

// 预设颜色列表 - 更鲜明的色调
const presetColors = ref([
  {
    name: '暖白光',
    value: '#FFE4CC'  // 温暖的米白色，适合日常自拍
  },
  {
    name: '冷白光',
    value: '#F0F8FF'  // 清爽的浅蓝白，适合清新风格
  },
  {
    name: '日光色',
    value: '#FFF5E6'  // 自然的日光色，最接近自然光
  },
  {
    name: '柔粉光',
    value: '#FFD1DC'  // 温柔的粉色，提亮肤色
  },
  {
    name: '香槟光',
    value: '#FFE8D6'  // 温暖的香槟色，提升气色
  },
  {
    name: '珍珠光',
    value: '#F5F5F5'  // 柔和的珍珠白，打造通透感
  },
  {
    name: '奶咖光',
    value: '#F5E6DB'  // 温暖的奶咖色，营造温馨感
  },
  {
    name: '月光色',
    value: '#E6F3FF'  // 清凉的月光色，创造清透感
  }
])

// 当前亮度值
const brightness = ref(50)
// 颜色深浅值
const colorValue = ref(50)
// 色轮选择的颜色
const wheelColor = ref('#FFFFFF')

// 当前选中的颜色索引
const currentColorIndex = ref(0)

// 当前颜色
const currentColor = computed(() => {
  if (mode.value === 'preset') {
    return presetColors.value[currentColorIndex.value].value
  } else if (mode.value === 'wheel') {
    return wheelColor.value
  } else if (mode.value === 'favorite') {
    // 如果是收藏模式，直接返回 wheelColor
    return wheelColor.value
  }
  return wheelColor.value
})

// 色轮相关状态
const wheelContext = ref(null)
const wheelSize = 200  // 从 160px 改为 200px
const wheelRadius = ref(wheelSize / 2)
const wheelCenterX = ref(wheelSize / 2)
const wheelCenterY = ref(wheelSize / 2)

// 初始化指针位置在中心
const pointerPosition = ref({
  x: wheelSize / 2,
  y: wheelSize / 2
})

// 修改色轮触摸处理函数
const handleWheelTouch = (e) => {
  const touch = e.touches[0]
  const query = uni.createSelectorQuery()
  
  query.select('#colorWheel').boundingClientRect(rect => {
    if (!rect) return
    
    let x = touch.clientX - rect.left
    let y = touch.clientY - rect.top
    
    // 限制在圆形范围内
    const centerX = wheelRadius.value
    const centerY = wheelRadius.value
    const dx = x - centerX
    const dy = y - centerY
    const distance = Math.sqrt(dx * dx + dy * dy)
    
    if (distance > wheelRadius.value) {
      const angle = Math.atan2(dy, dx)
      x = centerX + Math.cos(angle) * wheelRadius.value
      y = centerY + Math.sin(angle) * wheelRadius.value
    }
    
    // 更新指针位置
    pointerPosition.value = { x, y }
    
    // 计算颜色
    const angle = (Math.atan2(y - centerY, x - centerX) * 180 / Math.PI + 360) % 360
    const saturation = Math.min(100, (distance / wheelRadius.value) * 100)
    
    // 使用 tinycolor 生成颜色并更新 wheelColor
    const color = tinycolor({ h: angle, s: saturation, v: colorValue.value })
    wheelColor.value = color.toHexString()
  }).exec()
}

// ��理拾色器触摸
const handlePickerTouch = (e) => {
  e.stopPropagation()
  handleWheelTouch(e)
}

// 修改色轮绘制函数
const drawColorWheel = () => {
  const ctx = uni.createCanvasContext('colorWheel')
  const centerX = wheelCenterX.value
  const centerY = wheelCenterY.value
  const radius = wheelRadius.value

  // 清空画布
  ctx.clearRect(0, 0, wheelSize, wheelSize)

  // 绘制色相环
  for (let angle = 0; angle < 360; angle++) {
    const startAngle = (angle - 1) * Math.PI / 180
    const endAngle = (angle + 1) * Math.PI / 180

    const color = tinycolor({ h: angle, s: 100, v: 100 })
    ctx.beginPath()
    ctx.moveTo(centerX, centerY)
    ctx.arc(centerX, centerY, radius, startAngle, endAngle)
    ctx.closePath()
    ctx.fillStyle = color.toHexString()
    ctx.fill()
  }

  // 绘制白色渐变
  const gradient = ctx.createCircularGradient(centerX, centerY, radius)
  gradient.addColorStop(0, 'rgba(255, 255, 255, 1)')
  gradient.addColorStop(0.3, 'rgba(255, 255, 255, 0.8)')
  gradient.addColorStop(0.6, 'rgba(255, 255, 255, 0.3)')
  gradient.addColorStop(1, 'rgba(255, 255, 255, 0)')
  
  ctx.beginPath()
  ctx.arc(centerX, centerY, radius, 0, Math.PI * 2)
  ctx.fillStyle = gradient
  ctx.fill()

  ctx.draw(false)
}

// 修改颜色深浅变化处理
const handleColorValueChange = (e) => {
  const value = e.detail.value
  colorValue.value = value
  
  // 根据当前模式处理颜色深浅
  if (mode.value === 'preset') {
    // 预设色模式：调整当前选中颜色的深浅
    const color = tinycolor(presetColors.value[currentColorIndex.value].value)
    const hsv = color.toHsv()
    // 保持色相和饱和度不变，只调整明度
    const newColor = tinycolor({ h: hsv.h, s: hsv.s, v: value / 100 })
    presetColors.value[currentColorIndex.value].value = newColor.toHexString()
  } else {
    // 色轮模式：调整当前色轮颜色的深浅
    const color = tinycolor(wheelColor.value)
    const hsv = color.toHsv()
    // 保持色相和饱和度不变，只调整明度
    const newColor = tinycolor({ h: hsv.h, s: hsv.s, v: value / 100 })
    wheelColor.value = newColor.toHexString()
  }
}

// 处理颜色切换
const handleColorChange = (e) => {
  currentColorIndex.value = e.detail.current
}

// 修改亮度处理函数
const handleBrightnessChange = (e) => {
  const value = e.detail.value
  brightness.value = value
  // 调用系统API设置屏幕亮度
  uni.setScreenBrightness({
    value: value / 100,
    success: () => {
      // 可以添加成功回调
    },
    fail: (err) => {
      // 处理权限或其他错误
      uni.showToast({
        title: '设置亮度失败，请检查权限',
        icon: 'none'
      })
    }
  })
}

// 择颜色
const selectColor = (index) => {
  currentColorIndex.value = index
  // 使用颜色值
  const color = presetColors.value[index].value
}

// 修改初始化逻辑
onMounted(() => {
  // 获取屏幕亮度
  uni.getScreenBrightness({
    success: (res) => {
      brightness.value = Math.round(res.value * 100)
    }
  })

  // 如果初始模式是色轮，则绘制色轮
  if (mode.value === 'wheel') {
    nextTick(() => {
      setTimeout(drawColorWheel, 100)
    })
  }

  // 从本地存储加载收藏的颜色
  const savedColors = uni.getStorageSync('favoriteColors')
  if (savedColors) {
    favoriteColors.value = savedColors
  }
})

// 修改面板控制逻辑
const showPanel = ref(false)  // 将默认值改为 false

// 添加打开面板的方法
const openPanel = () => {
  showPanel.value = true
}

// 监听面板显示状态
watch(showPanel, (newValue) => {
  if (newValue) {
    // 每次显示面板时重置为预设色模式
    mode.value = 'preset'
    // 重置色轮相关状态
    wheelColor.value = '#FFFFFF'
    pointerPosition.value = {
      x: wheelSize / 2,
      y: wheelSize / 2
    }
  }
})

// 监听模式变化
watch(mode, (newMode) => {
  if (newMode === 'wheel' && showPanel.value) {
    // 切换到色轮模式时重新绘制
    nextTick(() => {
      setTimeout(() => {
        drawColorWheel()
      }, 50)
    })
  }
})

// 关闭面板
const closePanel = () => {
  showPanel.value = false
  // 重置色轮相关状态
  wheelColor.value = '#FFFFFF'
  pointerPosition.value = {
    x: wheelSize / 2,
    y: wheelSize / 2
  }
}

// 收藏颜色列表
const favoriteColors = ref([])

// 判断当前颜色是否已收藏
const isFavorite = computed(() => {
  const colorToCheck = mode.value === 'wheel' ? wheelColor.value : currentColor.value
  return favoriteColors.value.some(color => color.value === colorToCheck)
})

// 修改切换收藏状态的方法
const toggleFavorite = (e) => {
  e.stopPropagation() // 阻止事件冒泡，防止触发面板关闭
  
  const currentColorValue = mode.value === 'wheel' ? wheelColor.value : currentColor.value
  let currentColorName = '自定义颜色'
  
  // 根据不同模式设置颜色名称
  if (mode.value === 'preset') {
    currentColorName = presetColors.value[currentColorIndex.value].name
  } else if (mode.value === 'wheel') {
    // 为色轮模式生成更有意义的颜色名称
    const color = tinycolor(currentColorValue)
    const hsv = color.toHsv()
    currentColorName = `色轮 ${Math.round(hsv.h)}°`
  }

  // 检查是否已收藏
  const isAlreadyFavorited = favoriteColors.value.some(color => color.value === currentColorValue)

  if (isAlreadyFavorited) {
    // 取消收藏
    favoriteColors.value = favoriteColors.value.filter(
      color => color.value !== currentColorValue
    )
    uni.showToast({
      title: '已取消收藏',
      icon: 'none',
      duration: 1500
    })
  } else {
    // 添加收藏
    favoriteColors.value.push({
      name: currentColorName,
      value: currentColorValue,
      timestamp: Date.now()
    })
    uni.showToast({
      title: '已添加收藏',
      icon: 'none',
      duration: 1500
    })
  }

  // 保存到本地存储
  uni.setStorageSync('favoriteColors', favoriteColors.value)
}

// 添加删除收藏颜色的方法
const deleteFavoriteColor = (index) => {
  uni.showModal({
    title: '删除收藏',
    content: '确定要删除这个收藏的颜色吗？',
    success: (res) => {
      if (res.confirm) {
        favoriteColors.value.splice(index, 1)
        // 保存到本地存储
        uni.setStorageSync('favoriteColors', favoriteColors.value)
        uni.showToast({
          title: '已删除',
          icon: 'none'
        })
      }
    }
  })
}

// 修改选择收藏颜色的方法
const selectFavoriteColor = (color) => {
  // 更新当前显示的颜色
  wheelColor.value = color.value
  
  // 更新当前显示的颜色和深浅值
  const colorObj = tinycolor(color.value)
  const hsv = colorObj.toHsv()
  colorValue.value = Math.round(hsv.v * 100)
  
  // 更新背景颜色
  if (mode.value === 'favorite') {
    // 直接应用颜色，不关闭面板
    const newColor = tinycolor(color.value)
    wheelColor.value = newColor.toHexString()
    
    // 如果是在色轮模式下，更新指针位置
    if (mode.value === 'wheel') {
      const hsv = newColor.toHsv()
      const angle = hsv.h * Math.PI / 180
      const distance = hsv.s * wheelRadius.value / 100
      
      pointerPosition.value = {
        x: wheelCenterX.value + Math.cos(angle) * distance,
        y: wheelCenterY.value + Math.sin(angle) * distance
      }
    }
  }
}
</script>

<style scoped>
/* 添加遮罩层样式 */
.wrapper {
  width: 100%;
  height: 100vh;
  position: fixed;
  left: 0;
  top: 0;
}

.container {
  width: 100%;
  height: 100vh;
  transition: background-color 0.3s;
}

/* 修改毛玻璃效果 - 使用蓝色调 */
.glass-effect {
  background: rgba(64, 64, 64, 0.25);  /* 更深的灰色 */
  backdrop-filter: blur(25px) saturate(180%);
  -webkit-backdrop-filter: blur(25px) saturate(180%);
  border: 1px solid rgba(255, 255, 255, 0.12);
}

/* 修改面板样式 */
.color-panel {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  height: 50vh;
  z-index: 101;
  border-radius: 24px 24px 0 0;
  display: flex;
  flex-direction: column;
  animation: slideUp 0.3s cubic-bezier(0.16, 1, 0.3, 1);
  /* background: rgba(64, 64, 64, 0.25);   */
  backdrop-filter: blur(25px) saturate(180%);
  -webkit-backdrop-filter: blur(25px) saturate(180%);
}

/* 面板头部样式 */
.panel-header {
  padding: 4px 16px 8px; /* 增加左右内边距 */
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: center;
  border-bottom: 1px solid rgba(255, 255, 255, 0.06);
}

/* 顶部拖动条 */
.panel-header::after {
  content: '';
  width: 32px;
  height: 4px;
  background: rgba(255, 255, 255, 0.2);
  border-radius: 4px;
  margin-top: 8px;
}

/* 修改标签组样式，使其更圆润 */
.tab-group {
  margin-top: 8px;
  padding: 4px;
  background: rgba(255, 255, 255, 0.1);
  border-radius: 20px; /* 增加圆角 */
  display: inline-flex;
  gap: 4px;
  width: auto;
  min-width: 280px; /* 确保最小宽度 */
}

.tab-item {
  padding: 8px 16px; /* 调整内边距 */
  font-size: 13px;
  flex: 1;
  text-align: center;
  border-radius: 16px; /* 增加圆角 */
  color: rgba(255, 255, 255, 0.9);
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  position: relative;
  overflow: hidden;
}

.tab-item.active {
  background: rgba(255, 255, 255, 0.95);
  color: rgba(64, 64, 64, 0.9);
  font-weight: 500;
  transform: scale(1.02); /* 添加轻微的放大效果 */
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1); /* 添加轻微阴影 */
}

/* 添加悬停效果 */
.tab-item:not(.active):hover {
  background: rgba(255, 255, 255, 0.15);
}

/* 确保标签组在面板头部居中 */
.panel-header {
  padding: 4px 16px 8px; /* 增加左右内边距 */
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: center;
  border-bottom: 1px solid rgba(255, 255, 255, 0.06);
}

/* 预设色网格 */
.color-grid {
  flex: 1;
  overflow: hidden;
}

.color-grid-content {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  grid-template-rows: repeat(2, 1fr);
  gap: 12px;
  height: 200px;
  padding: 15px;
  position: relative;
}

.grid-item {
  position: relative;
  aspect-ratio: 1;
  width: 90%;
  margin: 0 auto;
}

.color-preview {
  width: 100%;
  height: 100%;
  border-radius: 50%;
  border: 1px solid rgba(255, 255, 255, 0.25);
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.06);
  transition: all 0.2s cubic-bezier(0.4, 0, 0.2, 1);
}

.grid-item.active .color-preview {
  transform: scale(1.05);
  border-color: rgba(255, 255, 255, 0.9);
  box-shadow: 0 3px 12px rgba(0, 0, 0, 0.12);
}

.color-name {
  position: absolute;
  bottom: -18px;
  left: 50%;
  transform: translateX(-50%);
  font-size: 11px;
  color: rgba(255, 255, 255, 0.9);
  white-space: nowrap;
  text-align: center;
}

/* 色轮区域 */
.wheel-mode {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 25px;
  padding: 15px 0;
  background: rgba(64, 64, 64, 0.25);  /* 统一背景色 */
  backdrop-filter: blur(25px) saturate(180%);
  -webkit-backdrop-filter: blur(25px) saturate(180%);
}

.wheel-container {
  width: 200px;
  height: 200px;
  position: relative;
  border-radius: 50%;
  overflow: hidden;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.12);
}

.wheel-canvas {
  width: 100%;
  height: 100%;
  display: block;
}

.color-picker {
  position: absolute;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  border: 2px solid #fff;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.2);
  transform: translate(-50%, -50%);
  pointer-events: none;
  z-index: 2;
}

/* 修改控制面板样式 */
.control-panel {
  padding: 16px;
  background: rgba(64, 64, 64, 0.3);
  backdrop-filter: blur(25px) saturate(180%);
  -webkit-backdrop-filter: blur(25px) saturate(180%);
  border-top: 1px solid rgba(255, 255, 255, 0.12);
  margin-top: auto;
  display: flex;
  flex-direction: row;
  gap: 16px;
}

.control-item {
  flex: 1;
  background: rgba(255, 255, 255, 0.1);
  border-radius: 12px;
  padding: 12px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.control-label {
  font-size: 13px;
  color: rgba(255, 255, 255, 0.9);
  text-align: center;
  letter-spacing: 0.5px;
  font-weight: 500;
}

/* 修改滑块样式 */
.control-slider {
  margin: 0;
  flex: 1;
}

/* 修改面板整体样式 - 使用灰色调 */
.glass-effect {
  background: rgba(128, 128, 128, 0.25);
  backdrop-filter: blur(20px) saturate(180%);
  -webkit-backdrop-filter: blur(20px) saturate(180%);
  border: 1px solid rgba(255, 255, 255, 0.12);
}

/* 修改标签激活状态样式 */
.tab-item.active {
  background: rgba(255, 255, 255, 0.9);
  color: rgba(64, 64, 64, 0.9);
  font-weight: 500;
}

/* 修改滑块轨道样式 */
.control-slider :deep(.uni-slider) {
  margin: 0;
}

/* 动画效果 */
@keyframes slideUp {
  from {
    transform: translateY(100%);
    opacity: 0;
  }
  to {
    transform: translateY(0);
    opacity: 1;
  }
}

/* 添加遮罩层样式 */
.mask {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.2);  /* 稍深的遮罩 */
  z-index: 100;
}

/* 确保面板在遮罩层之上 */
.color-panel {
  z-index: 101;
  /* ... 其他面板样式保持不变 ... */
}

/* 统一背景色变量 */
.panel-header,
.preset-mode,
.wheel-mode {
  background: rgba(64, 64, 64, 0.25);  /* 统一使用相同的背景色 */
  backdrop-filter: blur(25px) saturate(180%);
  -webkit-backdrop-filter: blur(25px) saturate(180%);
}

/* 预设色模式 */
.preset-mode {
  flex: 1;
  overflow: hidden;
}

/* 色轮模式 */
.wheel-mode {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 25px;
  padding: 15px 0;
}

/* 收藏按钮样式 */
.favorite-btn {
  position: absolute;
  top: 60px; /* 调整到标签栏下方 */
  right: 16px;
  width: 36px; /* 稍微调小一点 */
  height: 36px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 102;
  cursor: pointer;
  background: rgba(255, 255, 255, 0.1); /* 添加轻微背景 */
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  border: 1px solid rgba(255, 255, 255, 0.2);
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

.favorite-btn:active {
  transform: scale(0.95);
}

/* 调整图标样式 */
.iconfont {
  font-size: 20px; /* 稍微调小图标 */
  color: rgba(255, 255, 255, 0.8);
  transition: all 0.3s;
}

/* 优化激活状态样式 */
.icon-active {
  color: #FFD700;
  text-shadow: 0 0 8px rgba(255, 215, 0, 0.5);
  transform: scale(1.1);
}

/* 添加悬停效果 */
.favorite-btn:hover {
  background: rgba(255, 255, 255, 0.15);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

/* 确保收藏模式下的网格样式与预设模式一致 */
.favorite-mode {
  flex: 1;
  overflow: hidden;
  background: rgba(64, 64, 64, 0.25);
  backdrop-filter: blur(25px) saturate(180%);
  -webkit-backdrop-filter: blur(25px) saturate(180%);
}

/* 添加删除按钮样式 */
.delete-btn {
  position: absolute;
  top: -8px;
  right: -8px;
  width: 24px;
  height: 24px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  background: rgba(255, 255, 255, 0.9);
  cursor: pointer;
  z-index: 2;
  transition: all 0.2s;
}

.delete-btn:active {
  transform: scale(0.95);
}

.delete-icon {
  color: #FF4D4F;
  font-size: 18px;
  font-weight: bold;
  line-height: 1;
}

/* 调整网格项样式以适应删除按钮 */
.grid-item {
  position: relative;
  padding-top: 8px;
  padding-right: 8px;
}

/* 调整收藏图标的激活状态样式 */
.icon-active {
  color: #FFD700;
  text-shadow: 0 0 10px rgba(255, 215, 0, 0.5);
  transform: scale(1.1);
}
</style>
