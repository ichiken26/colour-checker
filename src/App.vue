<script setup>
import { ref, computed, watch } from 'vue'

// 色の状態管理
const colorInput = ref('')
const inputType = ref('hex')
const currentColor = ref('#3B82F6')

// 色変換関数
const hexToRgb = (hex) => {
  const result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex)
  return result ? {
    r: parseInt(result[1], 16),
    g: parseInt(result[2], 16),
    b: parseInt(result[3], 16)
  } : null
}

const rgbToHex = (r, g, b) => {
  return "#" + ((1 << 24) + (r << 16) + (g << 8) + b).toString(16).slice(1)
}

// RGBからYCbCrへの変換（ITU-R BT.601標準）
const rgbToYcbcr = (r, g, b) => {
  // 係数: CA = 0.299, CB = 0.114 (BT.601)
  const y = 0.299 * r + 0.587 * g + 0.114 * b
  const cb = 128 - 0.168736 * r - 0.331264 * g + 0.5 * b
  const cr = 128 + 0.5 * r - 0.418688 * g - 0.081312 * b
  
  return {
    y: Math.round(Math.max(16, Math.min(235, y))),
    cb: Math.round(Math.max(16, Math.min(240, cb))),
    cr: Math.round(Math.max(16, Math.min(240, cr)))
  }
}

// YCbCrからRGBへの変換（ITU-R BT.601標準）
const ycbcrToRgb = (y, cb, cr) => {
  // オフセット調整（限定レンジ: Y=16-235, Cb/Cr=16-240）
  const yAdj = y - 16
  const cbAdj = cb - 128
  const crAdj = cr - 128
  
  // 標準的なBT.601変換式
  const r = 1.164 * yAdj + 1.596 * crAdj
  const g = 1.164 * yAdj - 0.392 * cbAdj - 0.813 * crAdj
  const b = 1.164 * yAdj + 2.017 * cbAdj
  
  return {
    r: Math.max(0, Math.min(255, Math.round(r))),
    g: Math.max(0, Math.min(255, Math.round(g))),
    b: Math.max(0, Math.min(255, Math.round(b)))
  }
}

// 色の計算プロパティ
const colorInfo = computed(() => {
  if (!currentColor.value) return null
  
  const rgb = hexToRgb(currentColor.value)
  if (!rgb) return null
  
  const ycbcr = rgbToYcbcr(rgb.r, rgb.g, rgb.b)
  
  return {
    hex: currentColor.value.toUpperCase(),
    rgb: rgb,
    ycbcr: ycbcr
  }
})

// 入力値の検証と変換
const validateAndConvertColor = () => {
  if (!colorInput.value) return
  
  let rgb = null
  
  switch (inputType.value) {
    case 'hex':
      if (/^#?[0-9A-Fa-f]{6}$/.test(colorInput.value)) {
        const hex = colorInput.value.startsWith('#') ? colorInput.value : '#' + colorInput.value
        rgb = hexToRgb(hex)
        if (rgb) currentColor.value = hex
      }
      break
      
    case 'rgb':
      const rgbMatch = colorInput.value.match(/(\d+),\s*(\d+),\s*(\d+)/)
      if (rgbMatch) {
        const r = parseInt(rgbMatch[1])
        const g = parseInt(rgbMatch[2])
        const b = parseInt(rgbMatch[3])
        if (r >= 0 && r <= 255 && g >= 0 && g <= 255 && b >= 0 && b <= 255) {
          rgb = { r, g, b }
          currentColor.value = rgbToHex(r, g, b)
        }
      }
      break
      
    case 'ycbcr':
      const ycbcrMatch = colorInput.value.match(/(\d+),\s*(\d+),\s*(\d+)/)
      if (ycbcrMatch) {
        const y = parseInt(ycbcrMatch[1])
        const cb = parseInt(ycbcrMatch[2])
        const cr = parseInt(ycbcrMatch[3])
        // YCbCrの標準範囲: Y=16-235, Cb/Cr=16-240
        if (y >= 16 && y <= 235 && cb >= 16 && cb <= 240 && cr >= 16 && cr <= 240) {
          rgb = ycbcrToRgb(y, cb, cr)
          currentColor.value = rgbToHex(rgb.r, rgb.g, rgb.b)
        }
      }
      break
  }
}

// 入力タイプが変更された時の処理
watch(inputType, () => {
  colorInput.value = ''
})

// 色のコピー機能
const copyToClipboard = (text) => {
  navigator.clipboard.writeText(text)
}
</script>

<template>
  <div class="color-checker">
    <!-- ヘッダー -->
    <header class="header">
      <h1 class="title">Colour Checker</h1>
      <p class="subtitle">HEX、RGB、YCbCrの色の確認・変換ツール</p>
    </header>

    <!-- メインコンテンツ -->
    <main class="main">
      <!-- 色入力セクション -->
      <section class="input-section">
        <div class="input-group">
          <label class="input-label">色の入力形式</label>
          <div class="input-type-selector">
            <button 
              v-for="type in ['hex', 'rgb', 'ycbcr']" 
              :key="type"
              :class="['type-btn', { active: inputType === type }]"
              @click="inputType = type"
            >
              {{ type.toUpperCase() }}
            </button>
          </div>
        </div>

        <div class="input-group">
          <label class="input-label">
            {{ inputType === 'hex' ? 'HEX値 (例: #3B82F6)' : 
               inputType === 'rgb' ? 'RGB値 (例: 59, 130, 246)' : 
               'YCbCr値 (例: 120, 128, 150) - Y:16-235, Cb/Cr:16-240' }}
          </label>
          <div class="input-container">
            <input 
              v-model="colorInput"
              @input="validateAndConvertColor"
              :placeholder="inputType === 'hex' ? '#3B82F6' : 
                          inputType === 'rgb' ? '59, 130, 246' : 
                          '120, 128, 150'"
              class="color-input"
            />
            <button @click="validateAndConvertColor" class="convert-btn">
              変換
            </button>
          </div>
        </div>
      </section>

      <!-- 色プレビューセクション -->
      <section class="preview-section">
        <div class="color-preview" :style="{ backgroundColor: currentColor }">
          <div class="color-overlay">
            <div class="color-info">
              <h3>現在の色</h3>
              <p class="color-value">{{ currentColor }}</p>
            </div>
          </div>
        </div>
      </section>

      <!-- 色情報セクション -->
      <section class="info-section" v-if="colorInfo">
        <div class="info-grid">
          <!-- HEX情報 -->
          <div class="info-card">
            <h4>HEX</h4>
            <div class="info-content">
              <span class="info-value">{{ colorInfo.hex }}</span>
              <button @click="copyToClipboard(colorInfo.hex)" class="copy-btn">
                コピー
              </button>
            </div>
          </div>

          <!-- RGB情報 -->
          <div class="info-card">
            <h4>RGB</h4>
            <div class="info-content">
              <span class="info-value">
                {{ colorInfo.rgb.r }}, {{ colorInfo.rgb.g }}, {{ colorInfo.rgb.b }}
              </span>
              <button @click="copyToClipboard(`${colorInfo.rgb.r}, ${colorInfo.rgb.g}, ${colorInfo.rgb.b}`)" class="copy-btn">
                コピー
              </button>
            </div>
          </div>

          <!-- YCbCr情報 -->
          <div class="info-card">
            <h4>YCbCr</h4>
            <div class="info-content">
              <span class="info-value">
                {{ colorInfo.ycbcr.y }}, {{ colorInfo.ycbcr.cb }}, {{ colorInfo.ycbcr.cr }}
              </span>
              <button @click="copyToClipboard(`${colorInfo.ycbcr.y}, ${colorInfo.ycbcr.cb}, ${colorInfo.ycbcr.cr}`)" class="copy-btn">
                コピー
              </button>
            </div>
          </div>
        </div>
      </section>
    </main>
  </div>
</template>

<style scoped>
.color-checker {
  min-height: 100vh;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  padding: 2rem;
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
}

.header {
  text-align: center;
  margin-bottom: 3rem;
}

.title {
  font-size: 3rem;
  font-weight: 700;
  color: white;
  margin: 0 0 1rem 0;
  text-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.subtitle {
  font-size: 1.2rem;
  color: rgba(255, 255, 255, 0.9);
  margin: 0;
}

.main {
  max-width: 1200px;
  margin: 0 auto;
}

.input-section {
  background: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(10px);
  border-radius: 20px;
  padding: 2rem;
  margin-bottom: 2rem;
  box-shadow: 0 20px 40px rgba(0, 0, 0, 0.1);
}

.input-group {
  margin-bottom: 1.5rem;
}

.input-group:last-child {
  margin-bottom: 0;
}

.input-label {
  display: block;
  font-weight: 600;
  color: #374151;
  margin-bottom: 0.5rem;
  font-size: 1rem;
}

.input-type-selector {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 1rem;
}

.type-btn {
  padding: 0.75rem 1.5rem;
  border: 2px solid #e5e7eb;
  background: white;
  border-radius: 12px;
  font-weight: 600;
  color: #6b7280;
  cursor: pointer;
  transition: all 0.2s ease;
}

.type-btn:hover {
  border-color: #3b82f6;
  color: #3b82f6;
}

.type-btn.active {
  background: #3b82f6;
  border-color: #3b82f6;
  color: white;
}

.input-container {
  display: flex;
  gap: 1rem;
  align-items: center;
}

.color-input {
  flex: 1;
  padding: 1rem;
  border: 2px solid #e5e7eb;
  border-radius: 12px;
  font-size: 1rem;
  transition: border-color 0.2s ease;
}

.color-input:focus {
  outline: none;
  border-color: #3b82f6;
}

.convert-btn {
  padding: 1rem 2rem;
  background: #3b82f6;
  color: white;
  border: none;
  border-radius: 12px;
  font-weight: 600;
  cursor: pointer;
  transition: background-color 0.2s ease;
}

.convert-btn:hover {
  background: #2563eb;
}

.preview-section {
  margin-bottom: 2rem;
}

.color-preview {
  height: 300px;
  border-radius: 20px;
  position: relative;
  overflow: hidden;
  box-shadow: 0 20px 40px rgba(0, 0, 0, 0.1);
}

.color-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(135deg, rgba(0, 0, 0, 0.1), rgba(0, 0, 0, 0.3));
  display: flex;
  align-items: center;
  justify-content: center;
}

.color-info {
  text-align: center;
  color: white;
}

.color-info h3 {
  margin: 0 0 0.5rem 0;
  font-size: 1.5rem;
  font-weight: 600;
}

.color-value {
  font-size: 1.2rem;
  font-weight: 500;
  margin: 0;
  text-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
}

.info-section {
  background: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(10px);
  border-radius: 20px;
  padding: 2rem;
  box-shadow: 0 20px 40px rgba(0, 0, 0, 0.1);
}

.info-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 1.5rem;
}

.info-card {
  background: white;
  border-radius: 16px;
  padding: 1.5rem;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
  border: 1px solid #f3f4f6;
}

.info-card h4 {
  margin: 0 0 1rem 0;
  color: #374151;
  font-size: 1.1rem;
  font-weight: 600;
}

.info-content {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.info-value {
  flex: 1;
  font-family: 'Monaco', 'Menlo', monospace;
  font-size: 1rem;
  color: #1f2937;
  background: #f9fafb;
  padding: 0.5rem 0.75rem;
  border-radius: 8px;
  border: 1px solid #e5e7eb;
}

.copy-btn {
  padding: 0.5rem 1rem;
  background: #10b981;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  transition: background-color 0.2s ease;
}

.copy-btn:hover {
  background: #059669;
}

/* レスポンシブデザイン */
@media (max-width: 768px) {
  .color-checker {
    padding: 1rem;
  }
  
  .title {
    font-size: 2rem;
  }
  
  .input-container {
    flex-direction: column;
    gap: 0.5rem;
  }
  
  .convert-btn {
    width: 100%;
  }
  
  .info-grid {
    grid-template-columns: 1fr;
  }
  
  .info-content {
    flex-direction: column;
    align-items: stretch;
    gap: 0.5rem;
  }
  
  .copy-btn {
    width: 100%;
  }
}
</style>
