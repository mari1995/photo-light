<template>
  <view class="wrapper">
    <view 
      class="background-layer"
      @tap="togglePanel"
    ></view>
    
    <view 
      class="container" 
      :style="{ backgroundColor: currentColor }"
      @tap="openPanel"
    ></view>

    <view 
      class="color-panel" 
      v-if="showPanel" 
      @tap.stop
      @touchmove.stop.prevent
    >
      <view 
        class="favorite-btn"
        @tap.stop="toggleFavorite"
      >
        <text class="iconfont" :class="{ 'icon-active': isFavorite }">★</text>
      </view>

      <view class="panel-header">
        <view class="drag-handle"></view>
        
        <view class="tab-group">
          <view 
            class="tab-item" 
            :class="{ active: mode === 'preset' }"
            @tap="mode = 'preset'"
          >预设色</view>
          <view 
            class="tab-item" 
            :class="{ active: mode === 'wheel' }"
            @tap="mode = 'wheel'"
          >色轮</view>
          <view 
            class="tab-item" 
            :class="{ active: mode === 'favorite' }"
            @tap="mode = 'favorite'"
          >收藏</view>
          <view 
            class="tab-item" 
            :class="{ active: mode === 'image' }"
            @tap="mode = 'image'"
          >取图</view>
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
                class="delete-btn"
                @tap.stop="deleteFavoriteColor(index)"
              >
                <text class="delete-icon">×</text>
              </view>
            </view>
          </view>
        </scroll-view>
      </view>

      <view v-else-if="mode === 'image'" class="image-mode">
        <view class="image-content">
          <view class="image-container">
            <image 
              v-if="selectedImage"
              :src="selectedImage" 
              class="preview-image"
              mode="aspectFit"
            />
            <view v-else class="upload-placeholder" @tap="selectImage">
              <text class="iconfont">+</text>
              <text>点击选择图片</text>
            </view>
            <view v-if="selectedImage" class="reupload-btn" @tap="selectImage">
              <text class="iconfont">↺</text>
            </view>
          </view>
          
          <view v-if="selectedImage" class="image-controls">
            <view class="detected-color">
              <view 
                class="color-preview" 
                :style="{ backgroundColor: detectedColor }"
              ></view>
              <text class="color-value">{{ detectedColor }}</text>
            </view>
            <button class="pick-btn" @tap="selectImage">重新选择</button>
          </view>
        </view>
      </view>

      <view class="control-panel">
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

    <canvas 
      canvas-id="imageCanvas" 
      style="position: fixed; left: -999px; width: 100px; height: 100px;"
    ></canvas>

    <view 
      class="camera-btn glass-effect"
      @tap.stop="toggleCamera"
    >
      <text class="iconfont">📷</text>
    </view>

    <view 
      class="camera-preview" 
      v-if="showCamera"
      :style="{
        left: `${cameraPosition.x}px`,
        top: `${cameraPosition.y}px`
      }"
    >
      <view 
        class="camera-drag-handle"
        @touchstart.stop="handleCameraDragStart"
        @touchmove.stop="handleCameraDragMove"
        @touchend.stop="handleCameraDragEnd"
        @touchcancel.stop="handleCameraDragEnd"
      ></view>

      <camera
        class="camera"
        device-position="front"
        flash="off"
        @error="handleCameraError"
      ></camera>
      
      <view class="camera-controls glass-effect">
        <view 
          class="capture-btn"
          @tap.stop="takePhoto"
        >
          <view class="capture-btn-inner"></view>
        </view>
        <view 
          class="close-btn"
          @tap.stop="toggleCamera"
        >
          <text class="iconfont">✕</text>
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
    value: '#F0F8FF'  // 爽的浅蓝，适合清风格
  },
  {
    name: '日光色',
    value: '#FFF5E6'  // 然的日光色，最接近自然光
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
    value: '#F5F5F5'  // 柔和白，打造通透感
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
  } else if (mode.value === 'image') {
    return detectedColor.value
  }
  return wheelColor.value
})

// 色轮相关状态
const wheelContext = ref(null)
const wheelSize = 200  // 从 160px 改为 200px
const wheelRadius = ref(wheelSize / 2)
const wheelCenterX = ref(wheelSize / 2)
const wheelCenterY = ref(wheelSize / 2)

// 初化指针位置在中心
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

// 理拾色器触摸
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

// 修改颜色深浅变处理
const handleColorValueChange = (e) => {
  const value = e.detail.value
  colorValue.value = value
  
  // 根据当前模式立即更新颜色
  if (mode.value === 'preset') {
    const color = tinycolor(presetColors.value[currentColorIndex.value].value)
    const hsv = color.toHsv()
    const newColor = tinycolor({ h: hsv.h, s: hsv.s, v: value / 100 })
    presetColors.value[currentColorIndex.value].value = newColor.toHexString()
    wheelColor.value = newColor.toHexString()  // 直接更新当前颜色
  } else {
    const color = tinycolor(wheelColor.value)
    const hsv = color.toHsv()
    const newColor = tinycolor({ h: hsv.h, s: hsv.s, v: value / 100 })
    wheelColor.value = newColor.toHexString()  // 直接更新当前颜色
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

// 修改预设色选择方法
const selectColor = (index) => {
  currentColorIndex.value = index
  // 直接使用选中的颜色值更新当前颜色
  const color = presetColors.value[index].value
  wheelColor.value = color  // 更新 wheelColor
}

// 修改初始化逻辑
onMounted(() => {
  // 获取屏幕亮度
  uni.getScreenBrightness({
    success: (res) => {
      brightness.value = Math.round(res.value * 100)
    }
  })

  // 默认打开面板时初始化色轮
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

// 修改面板控制逻辑，默认显示
const showPanel = ref(true)

// 修改面板开关方法
const togglePanel = () => {
  showPanel.value = !showPanel.value
  
  // 如果是打开面板，并且当前是色轮模式，重新初始化色轮
  if (showPanel.value && mode.value === 'wheel') {
    nextTick(() => {
      setTimeout(() => {
        drawColorWheel()
        // 恢复之前选择的颜色位置
        const color = tinycolor(wheelColor.value)
        const hsv = color.toHsv()
        const angle = hsv.h * Math.PI / 180
        const distance = hsv.s * wheelRadius.value / 100
        
        // 更新指针位置
        pointerPosition.value = {
          x: wheelCenterX.value + Math.cos(angle) * distance,
          y: wheelCenterY.value + Math.sin(angle) * distance
        }
      }, 50)
    })
  }
}

// 监听模式变化
watch(mode, (newMode) => {
  if (newMode === 'wheel' && showPanel.value) {
    nextTick(() => {
      setTimeout(() => {
        drawColorWheel()
        // 恢复之前选择的颜色位置
        const color = tinycolor(wheelColor.value)
        const hsv = color.toHsv()
        const angle = hsv.h * Math.PI / 180
        const distance = hsv.s * wheelRadius.value / 100
        
        // 更新指针位置
        pointerPosition.value = {
          x: wheelCenterX.value + Math.cos(angle) * distance,
          y: wheelCenterY.value + Math.sin(angle) * distance
        }
      }, 50)
    })
  }
})

// 收藏色列表
const favoriteColors = ref([])

// 判断当前颜色是否已收藏
const isFavorite = computed(() => {
  const colorToCheck = mode.value === 'wheel' ? wheelColor.value : currentColor.value
  return favoriteColors.value.some(color => color.value === colorToCheck)
})

// 修改切换收藏状态方法
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
    content: '确定要删除这个收的颜色吗？',
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

// 修改收藏色选择方法
const selectFavoriteColor = (color) => {
  // 更新当前显示的颜色
  wheelColor.value = color.value
  
  // 更新深浅值
  const colorObj = tinycolor(color.value)
  const hsv = colorObj.toHsv()
  colorValue.value = Math.round(hsv.v * 100)
}

// 图片相关状态
const selectedImage = ref('')
const detectedColor = ref('')

// 在 script setup 中添加获取图片颜色的函数
const getImageColor = (imagePath) => {
  return new Promise((resolve, reject) => {
    const ctx = uni.createCanvasContext('imageCanvas')
    
    // 在画布中绘制图片
    ctx.drawImage(imagePath, 0, 0, 100, 100)  // 缩小寸以提高性能
    ctx.draw(false, () => {
      // 获取中心点的颜色
      uni.canvasGetImageData({
        canvasId: 'imageCanvas',
        x: 45,  // 取中心点附近的位置
        y: 45,
        width: 10,
        height: 10,
        success: (res) => {
          // 计算平均颜色
          let r = 0, g = 0, b = 0
          const data = res.data
          const total = data.length / 4
          
          for (let i = 0; i < data.length; i += 4) {
            r += data[i]
            g += data[i + 1]
            b += data[i + 2]
          }
          
          r = Math.round(r / total)
          g = Math.round(g / total)
          b = Math.round(b / total)
          
          resolve(`rgb(${r}, ${g}, ${b})`)
        },
        fail: reject
      })
    })
  })
}

// 修改图片颜色提取方法
const selectImage = () => {
  uni.chooseImage({
    count: 1,
    sourceType: ['album', 'camera'],
    success: async (res) => {
      selectedImage.value = res.tempFilePaths[0]
      try {
        const color = await getImageColor(res.tempFilePaths[0])
        detectedColor.value = color
        // 直接更新当前颜色
        wheelColor.value = tinycolor(color).toHexString()
      } catch (error) {
        uni.showToast({
          title: '颜色提取失败',
          icon: 'none'
        })
      }
    }
  })
}

// 分享给朋友
const onShareAppMessage = () => {
  return {
    title: '智能补光神器',
    path: '/pages/index/index',
    success(res) {
      uni.showToast({
        title: '分享成功',
        icon: 'success'
      })
    },
    fail(res) {
      uni.showToast({
        title: '分享失败',
        icon: 'none'
      })
    }
  }
}

// 分享到朋友圈
const onShareTimeline = () => {
  return {
    title: '智能补光神器',
    query: '',
    success(res) {
      uni.showToast({
        title: '分享成功',
        icon: 'success'
      })
    },
    fail(res) {
      uni.showToast({
        title: '分享失败',
        icon: 'none'
      })
    }
  }
}

// 相机相关状态
const showCamera = ref(false)
const cameraContext = ref(null)
const cameraDevicePosition = ref('front') // 'front' 或 'back'

// 切换相机显示
const toggleCamera = () => {
  showCamera.value = !showCamera.value
  if (showCamera.value) {
    // 初始化相机上下文
    nextTick(() => {
      cameraContext.value = uni.createCameraContext()
    })
  }
}

// 切换前后摄像头
const switchCamera = () => {
  cameraDevicePosition.value = cameraDevicePosition.value === 'front' ? 'back' : 'front'
}

// 拍照
const takePhoto = () => {
  if (!cameraContext.value) {
    cameraContext.value = uni.createCameraContext()
  }
  
  cameraContext.value.takePhoto({
    quality: 'high',
    success: (res) => {
      // 保存照片到相册
      uni.saveImageToPhotosAlbum({
        filePath: res.tempImagePath,
        success: () => {
          uni.showToast({
            title: '已保存到相册',
            icon: 'success',
            duration: 1500
          })
        },
        fail: () => {
          uni.showToast({
            title: '保存失败',
            icon: 'none',
            duration: 1500
          })
        }
      })
    },
    fail: () => {
      uni.showToast({
        title: '拍照失败',
        icon: 'none',
        duration: 1500
      })
    }
  })
}

// 处理相机错误
const handleCameraError = (e) => {
  uni.showToast({
    title: '相机启动失败，请检查权限',
    icon: 'none'
  })
  showCamera.value = false
}

// 添加相机预览位置状态
const cameraPosition = ref({ x: 100, y: 100 }) // 初始位置
const isDraggingCamera = ref(false)
const dragOffset = ref({ x: 0, y: 0 })

// 添加相机拖动处理函数
const handleCameraDragStart = (e) => {
  isDraggingCamera.value = true
  const touch = e.touches[0]
  dragOffset.value = {
    x: touch.clientX - cameraPosition.value.x,
    y: touch.clientY - cameraPosition.value.y
  }
}

const handleCameraDragMove = (e) => {
  if (!isDraggingCamera.value) return
  
  const touch = e.touches[0]
  
  // 计算新位置
  let newX = touch.clientX - dragOffset.value.x
  let newY = touch.clientY - dragOffset.value.y
  
  // 获取屏幕尺寸
  const screenWidth = uni.getSystemInfoSync().windowWidth
  const screenHeight = uni.getSystemInfoSync().windowHeight
  
  // 限制预览框不超出屏幕边界，修改为200px宽度
  newX = Math.max(0, Math.min(newX, screenWidth - 200))
  newY = Math.max(0, Math.min(newY, screenHeight - 200))
  
  // 更新位置
  cameraPosition.value = { x: newX, y: newY }
}

const handleCameraDragEnd = () => {
  isDraggingCamera.value = false
}
</script>

<style scoped>
.background-layer {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 99;  /* 确保在 container 之上，panel 之下 */
}

.container {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 1;
}

.color-panel {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  height: 50vh;
  display: flex;
  flex-direction: column;
  background: #ffffff;
  border-radius: 20px 20px 0 0;
  z-index: 100;
}

/* 修改控制面板样式 */
.control-panel {
  position: absolute; /* 改为绝对定位 */
  bottom: 0; /* 固定在底部 */
  left: 0;
  right: 0;
  padding: 12px 16px;
  background: #ffffff;
  border-top: 0.5px solid rgba(0, 0, 0, 0.05);
  display: flex;
  flex-direction: row;
  gap: 12px;
  z-index: 3;
}

/* 修改控制项样式 */
.control-item {
  flex: 1;
  display: flex;
  flex-direction: row;
  align-items: center;
  gap: 10px; /* 稍微增加间距 */
  background: #f8f8f8;
  border-radius: 10px;
  padding: 8px 14px; /* 调整内边距 */
}

/* 修改控制标签样式 */
.control-label {
  font-size: 15px; /* 改为15px */
  color: #666;
  min-width: 30px;
  font-weight: 500; /* 稍微加粗一点 */
}

/* 修改滑块样式 */
.control-slider {
  flex: 1;
  height: 24px;
  margin: 0;
}

/* 调整滑块组件样式 */
.control-slider :deep(.uni-slider) {
  margin: 0;
}

.control-slider :deep(.uni-slider-handle) {
  width: 16px;
  height: 16px;
  background: #ffffff;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.control-slider :deep(.uni-slider-track) {
  height: 2px;
  background: #ff2442 !important;
}

/* 修改内容区域样式，为底部控制面板留出空间 */
.preset-mode,
.wheel-mode,
.favorite-mode,
.image-mode {
  flex: 1;
  overflow: hidden;
  padding-bottom: 60px; /* 为底部控制面板留出空间 */
  min-height: 0;
}

/* 修改网格内容样式 */
.color-grid {
  height: 100%;
  overflow-y: auto;
  padding: 16px;
  padding-bottom: 80px; /* 确保内容不被控制面板遮挡 */
}

/* 修改色轮容器样式 */
.wheel-mode {
  padding: 12px 0;
}

.image-mode {
  padding: 16px;
}

/* 移除所有模式的灰色背景 */
.preset-mode,
.wheel-mode,
.favorite-mode,
.image-mode {
  background: #ffffff;
  flex: 1;
  overflow: hidden;
  padding-bottom: 0;  /* 移除底部内边距 */
  min-height: 0;  /* 允许内容区域被压缩 */
}

/* 修改控制面板样式 */
.control-panel {
  padding: 16px;
  background: #ffffff;
  border-top: 0.5px solid rgba(0, 0, 0, 0.05);
  margin-top: 0;  /* 移除上边距 */
  display: flex;
  flex-direction: row;
  gap: 20px;
  position: relative;  /* 添加相对定位 */
  z-index: 3;  /* 确保在其他内容之上 */
}

/* 修改各个模式的内容区域样式，减小高度给控制面板留出空间 */
.preset-mode,
.wheel-mode,
.favorite-mode,
.image-mode {
  flex: 1;
  overflow: hidden;
  padding-bottom: 0;
  min-height: 0;
  max-height: calc(100% - 140px);  /* 减小内容区域高度，给控制面板留出空间 */
}

/* 修改网格内容高度 */
.color-grid-content {
  padding: 16px;
  height: auto;  /* 改为自适应高度 */
}

/* 修改色轮容器样式 */
.wheel-mode {
  padding: 12px 0;
}

/* 修改图片模式样式 */
.image-mode {
  padding: 16px;
}

/* 修改面板头部样式 */
.panel-header {
  padding: 4px 16px 8px;
  background: #ffffff;
  border-bottom: 0.5px solid rgba(0, 0, 0, 0.05);
  display: flex;
  flex-direction: column;
  align-items: center;
  position: relative; /* 添加相对定位 */
  min-height: 70px; /* 确保有足够空间 */
}

/* 修改标签组样式 */
.tab-group {
  margin-top: 24px; /* 增加上边距，为收藏按钮留出空间 */
  margin-bottom: 8px;
  padding: 3px;
  background: rgba(0, 0, 0, 0.03);
  border-radius: 16px;
  display: inline-flex;
  gap: 3px;
  width: auto;
  min-width: 280px;
  justify-content: center;
}

/* 修改标签项样式 */
.tab-item {
  flex: 1;
  padding: 6px 16px; /* 调整内边距 */
  font-size: 13px;
  color: #666;
  text-align: center;
  border-radius: 14px;
  transition: all 0.3s;
  min-width: 60px; /* 设置最小宽度确保均匀分布 */
}

/* 拖动手柄样式 */
.drag-handle {
  width: 32px;
  height: 3px;
  background: #e5e5e5;
  border-radius: 2px;
  margin: 6px auto; /* 使用 auto 边距实现水平居中 */
}

/* 修改色轮区域样式 */
.wheel-mode {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 25px;
  padding: 12px 0;  /* 减小内边距 */
  background: #ffffff;  /* 改为白色背景 */
}

/* 修改收藏模样式 */
.favorite-mode {
  flex: 1;
  overflow: hidden;
  background: #ffffff;  /* 改为纯白背景 */
}

/* 修改图片模式样式 */
.image-mode {
  flex: 1;
  padding: 16px;
  background: #ffffff;
  overflow: hidden;
}

.image-content {
  display: flex;
  flex-direction: row;  /* 改为横向布局 */
  gap: 16px;
  padding: 16px;
  height: 180px;  /* 固定高度 */
}

.image-container {
  flex: 1;  /* 占据左侧空间 */
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 12px;
  overflow: hidden;
  background: #f8f8f8;
  border: 1px dashed #e5e5e5;
  transition: all 0.3s;
  position: relative;
}

.preview-image {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

/* 修改颜色提取区域样式 */
.image-controls {
  width: 120px;  /* 固定宽度 */
  display: flex;
  flex-direction: column;  /* 改为纵向布局 */
  gap: 12px;
  justify-content: center;
}

.detected-color {
  display: flex;
  flex-direction: column;  /* 改为纵向布局 */
  align-items: center;
  gap: 8px;
  padding: 12px;
  border-radius: 12px;
  background: #f8f8f8;
}

.detected-color .color-preview {
  width: 60px;
  height: 60px;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.detected-color .color-value {
  color: #333;
  font-size: 12px;
  text-align: center;
}

/* 修改重新选择按钮样式 */
.pick-btn {
  height: 36px;
  width: 100%;
  font-size: 14px;
  background: #ff2442;
  color: #fff;
  border-radius: 18px;
  display: flex;
  align-items: center;
  justify-content: center;
}

/* 修改颜色预览样式 */
.color-preview {
  border: 1px solid rgba(0, 0, 0, 0.08);  /* 更淡的边框 */
}

/* 修改标签组样式 */
.tab-group {
  background: #ffffff;
}

/* 修改收藏按钮样式 */
.favorite-btn {
  position: absolute;
  top: 16px;
  right: 16px;
  width: 36px;
  height: 36px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 102;
  background: rgba(0, 0, 0, 0.03);
  transition: all 0.3s;
}

.favorite-btn:active {
  transform: scale(0.95);
}

/* 修改收藏图标样式 */
.favorite-btn .iconfont {
  font-size: 20px;
  color: #666;
  transition: all 0.3s;
}

.favorite-btn .icon-active {
  color: #FFD700;
  text-shadow: 0 0 8px rgba(255, 215, 0, 0.5);
  transform: scale(1.1);
}

/* 顶部拖动条 */
.drag-handle {
  width: 40px;
  height: 4px;
  background: #e5e5e5;
  border-radius: 2px;
  margin: 8px auto;
}

/* 标签组样式 */
.tab-group {
  display: flex;
  padding: 0 12px;
  margin: 8px 0 0 0;
  position: relative;
  white-space: nowrap;
  overflow-x: auto;
  -webkit-overflow-scrolling: touch;
  scrollbar-width: none; /* Firefox */
}

.tab-group::-webkit-scrollbar {
  display: none; /* Chrome Safari */
}

/* 标签项样式 */
.tab-item {
  position: relative;
  padding: 5px;
  /* padding: 12px 0px; */
  font-size: 15px;
  color: #999;
  flex-shrink: 0;
  transition: all 0.3s;
}

/* 活动标签样式 */
.tab-item.active {
  color: #000;
  font-weight: 600;
}

/* 活动标签底部指示器 */
.tab-item.active::after {
  content: '';
  position: absolute;
  bottom: 0;
  left: 16px;
  right: 16px;
  height: 3px;
  background: #ff2442;
  border-radius: 3px;
}

/* 移除玻璃拟态效果 */
.glass-effect {
  background: none;
  backdrop-filter: none;
  -webkit-backdrop-filter: none;
  border: none;
}

/* 统一网格布局样式 */
.color-grid {
  flex: 1;
  overflow: hidden;
  padding: 20px;
}

.color-grid-content {
  display: grid;
  grid-template-columns: repeat(4, 1fr);  /* 固定4列 */
  gap: 20px;  /* 统一间距 */
  padding: 0 12px;
}

/* 统一网格项样式 */
.grid-item {
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  width: 100%;
  max-width: 70px;  /* 限制最大宽度 */
  margin: 0 auto;
}

/* 统一颜色预览样式 */
.color-preview {
  width: 100%;
  padding-bottom: 100%;  /* 保持正方形比例 */
  position: relative;
  border-radius: 12px;  /* 统一使用圆角矩形 */
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
  transition: all 0.2s;
}

/* 统一颜色名称样式 */
.color-name {
  font-size: 12px;
  color: #666;
  text-align: center;
  width: 100%;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  margin-top: 4px;
}

/* 统一激活状态样式 */
.grid-item.active .color-preview {
  transform: scale(1.05);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.grid-item.active .color-name {
  color: #333;
  font-weight: 500;
}

/* 收藏模式特有的删除按钮样式 */
.favorite-mode .grid-item {
  position: relative;
}

.delete-btn {
  position: absolute;
  top: -6px;
  right: -6px;
  width: 20px;
  height: 20px;
  border-radius: 10px;
  background: #fff;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.1);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2;
}

.delete-icon {
  color: #ff2442;
  font-size: 14px;
  font-weight: bold;
}

/* 空状态提示 */
.empty-state {
  text-align: center;
  padding: 40px 0;
  color: #999;
  font-size: 14px;
}

/* 色轮区域 */
.wheel-mode {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 25px;
  padding: 12px 0;  /* 减小内边距 */
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
  background: #ffffff;
  border-top: 0.5px solid rgba(0, 0, 0, 0.05);
  margin-top: 0;  /* 移除上边距 */
  display: flex;
  flex-direction: row;
  gap: 20px;
  position: relative;  /* 添加相对定位 */
  z-index: 3;  /* 确保在其他内容之上 */
}

.control-item {
  flex: 1;
  background: #f8f8f8;  /* 浅灰背景保留，因为是控制项 */
  border-radius: 12px;
  padding: 12px;
}

.control-label {
  font-size: 14px;
  color: #333;
  text-align: center;
  font-weight: 500;
}

/* 修改滑块样式 */
.control-slider {
  margin: 0;
}

.control-slider :deep(.uni-slider) {
  margin: 0;
}

.control-slider :deep(.uni-slider-handle) {
  width: 24px;
  height: 24px;
  background: #ffffff;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.1);
}

.control-slider :deep(.uni-slider-track) {
  background: #ff2442 !important;  /* 强制使用小红书红色 */
}

/* 修改面板整体样式 - 使用灰色调 */
.glass-effect {
  background: rgba(128, 128, 128, 0.25);
  backdrop-filter: blur(20px) saturate(180%);
  -webkit-backdrop-filter: blur(20px) saturate(180%);
  border: 1px solid rgba(255, 255, 255, 0.12);
}

/* 改标签激活状态样式 */
.tab-item.active {
  background: rgba(255, 255, 255, 0.9);
  color: rgba(64, 64, 64, 0.9);
  font-weight: 500;
}

/* 改滑块轨道样式 */
.control-slider :deep(.uni-slider) {
  margin: 0;
}

/* 动画效 */
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

/* 确保其他相关样式使用纯色 */
.preset-mode,
.wheel-mode {
  background: #ffffff;
}

/* 预设色模式 */
.preset-mode {
  flex: 1;
  overflow: hidden;
  padding: 16px;
  background: #ffffff;
}

.color-grid {
  height: 100%;
  overflow-y: auto;  /* 允许垂直滚动 */
  padding: 0;
}

.color-grid-content {
  display: grid;
  grid-template-columns: repeat(4, 1fr);  /* 保持4列 */
  gap: 16px;  /* 减小间距 */
  padding: 16px;
  height: auto;  /* 移除固定高度，改为自适应 */
}

.grid-item {
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  width: 100%;
  max-width: 65px;  /* 稍微减小最大宽度 */
  margin: 0 auto;
}

.color-preview {
  width: 100%;
  padding-bottom: 100%;  /* 保持正方形比例 */
  position: relative;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
  transition: all 0.2s;
}

.color-name {
  font-size: 12px;
  color: #666;
  text-align: center;
  width: 100%;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  margin-top: 4px;
}

/* 色轮模式 */
.wheel-mode {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 25px;
  padding: 12px 0;  /* 减小内边距 */
}

/* 收藏按钮样式 */
.favorite-btn {
  position: absolute;
  top: 16px;
  right: 16px;
  width: 32px;
  height: 32px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 102;
  background: #f8f8f8;
  border: none;
  transition: all 0.2s;
}

.iconfont {
  font-size: 18px;
  color: #666;
}

.icon-active {
  color: #ff2442;
  text-shadow: none;
  transform: scale(1.1);
}

/* 添加悬停效果 */
.favorite-btn:hover {
  background: #f5f5f5;  /* 改为浅灰色 */
}

/* 确保收藏模式下的网格样式与预设模式一致 */
.favorite-mode {
  flex: 1;
  overflow: hidden;
  background: #ffffff;  /* 改为纯白背景 */
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
  background: #ffffff;
  cursor: pointer;
  z-index: 2;
  transition: all 0.2s;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.08);
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
}

/* 调整收藏标的激活状态样式 */
.icon-active {
  color: #FFD700;
  text-shadow: 0 0 10px rgba(255, 215, 0, 0.5);
  transform: scale(1.1);
}

/* 修改图片模式相关样式 */
.image-mode {
  flex: 1;
  padding: 16px;
  background: #ffffff;
  overflow: hidden;
}

.image-content {
  display: flex;
  flex-direction: row;  /* 改为横向布局 */
  gap: 16px;
  padding: 16px;
  height: 180px;  /* 固定高度 */
}

.image-container {
  flex: 1;  /* 占据左侧空间 */
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 12px;
  overflow: hidden;
  background: #f8f8f8;
  border: 1px dashed #e5e5e5;
  transition: all 0.3s;
  position: relative;
}

.preview-image {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.upload-placeholder {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 12px;
  transition: all 0.3s;
}

.upload-placeholder:active {
  background: #f5f5f5;
}

.upload-placeholder .iconfont {
  font-size: 28px;
  color: #999;
  margin-bottom: 4px;
}

.upload-placeholder text:not(.iconfont) {
  color: #666;
  font-size: 14px;
  font-weight: 500;
}

/* 修改颜色提取区域样式 */
.image-controls {
  width: 120px;  /* 固定宽度 */
  display: flex;
  flex-direction: column;  /* 改为纵向布局 */
  gap: 12px;
  justify-content: center;
}

.detected-color {
  display: flex;
  flex-direction: column;  /* 改为纵向布局 */
  align-items: center;
  gap: 8px;
  padding: 12px;
  border-radius: 12px;
  background: #f8f8f8;
}

.detected-color .color-preview {
  width: 60px;
  height: 60px;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.detected-color .color-value {
  color: #333;
  font-size: 12px;
  text-align: center;
}

/* 修改重新选择按钮样式 */
.pick-btn {
  height: 36px;
  width: 100%;
  font-size: 14px;
  background: #ff2442;
  color: #fff;
  border-radius: 18px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.pick-btn:active {
  opacity: 0.9;
}

.pick-btn::before {
  content: '';  /* 移除箭头图标 */
}

.reupload-btn {
  position: absolute;
  top: 8px;
  right: 8px;
  width: 32px;
  height: 32px;
  border-radius: 16px;
  background: rgba(255, 255, 255, 0.9);
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  transition: all 0.2s;
  z-index: 2;
}

.reupload-btn:active {
  transform: scale(0.95);
}

.reupload-btn .iconfont {
  font-size: 18px;
  color: #333;
}

/* 相机按钮样式 */
.camera-btn {
  position: fixed;
  top: 40px;
  left: 20px;
  width: 40px;
  height: 40px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 99;
  background: rgba(255, 255, 255, 0.2);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  border: 1px solid rgba(255, 255, 255, 0.3);
  transition: all 0.3s;
}

.camera-btn:active {
  transform: scale(0.95);
}

/* 相机预览容器 */
.camera-preview {
  position: fixed;
  width: 200px;
  height: 200px;
  border-radius: 16px;
  overflow: hidden;
  z-index: 99;
  background: #000;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.2);
}

/* 修改拖动手柄样式 */
.camera-drag-handle {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: 16px;
  background: rgba(0, 0, 0, 0.3);
  z-index: 2;
}

/* 调整相机视图大小 */
.camera {
  width: 100%;
  height: calc(100% - 16px);
  margin-top: 16px;
}

/* 修改控制按钮组样式 */
.camera-controls {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: rgba(0, 0, 0, 0.3);
  padding: 0 16px;
}

/* 修改拍照按钮样式 */
.capture-btn {
  width: 28px;
  height: 28px;
  border-radius: 50%;
  background: #fff;
  display: flex;
  align-items: center;
  justify-content: center;
}

.capture-btn-inner {
  width: calc(100% - 4px);
  height: calc(100% - 4px);
  border-radius: 50%;
  background: #fff;
  border: 1px solid rgba(0, 0, 0, 0.1);
}

/* 修改关闭按钮样式 */
.close-btn {
  width: 24px;
  height: 24px;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.8);
  display: flex;
  align-items: center;
  justify-content: center;
}

.close-btn .iconfont {
  color: #333;
  font-size: 14px;
}

/* 添加遮罩层 */
.camera-mask {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  z-index: 98;
  backdrop-filter: blur(5px);
  -webkit-backdrop-filter: blur(5px);
}
</style>
