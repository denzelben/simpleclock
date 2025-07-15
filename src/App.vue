<template>
  <div class="clock-container" @mousemove="handleMouseMove">
    <div class="color-picker-container">
      <button @click="toggleColorPicker" class="color-picker-toggle">
        {{ showColorPicker ? '▲' : '▼' }}
      </button>
      <div v-if="showColorPicker" class="color-picker">
        <input 
          type="color" 
          v-model="color1" 
          @input="updateGradient"
          class="color-input"
        >
        <input 
          type="color" 
          v-model="color2" 
          @input="updateGradient"
          class="color-input"
        >
      </div>
    </div>
    <div class="watermark">
      simple silly project by db
    </div>
    <!-- Fallback time display -->
    <div v-if="!threeJSReady" class="fallback-clock">
      <div class="fallback-date">{{ currentDate }}</div>
      <div class="fallback-time">{{ currentTime }}</div>
      <div class="fallback-year">{{ currentYear }}</div>
    </div>
    <div ref="clockContainer" class="clock-canvas"></div>
  </div>
</template>

<script>
import { ref, onMounted, onUnmounted } from 'vue'
import * as THREE from 'three'

export default {
  name: 'App',
  setup() {
    const clockContainer = ref(null)
    const color1 = ref('#667eea')
    const color2 = ref('#764ba2')
    const showColorPicker = ref(false)
    const threeJSReady = ref(false)
    const currentTime = ref('')
    const currentDate = ref('')
    const currentYear = ref('')
    let scene, camera, renderer, clockText
    let animationId
    let mouseX = 0, mouseY = 0
    let bubbles = []

    const toggleColorPicker = () => {
      showColorPicker.value = !showColorPicker.value
    }

    const updateGradient = () => {
      const root = document.documentElement
      root.style.setProperty('--gradient-color1', color1.value)
      root.style.setProperty('--gradient-color2', color2.value)
    }

    const updateFallbackTime = () => {
      const now = new Date()
      currentTime.value = now.toLocaleTimeString('en-US', { 
        hour12: false,
        hour: '2-digit',
        minute: '2-digit',
        second: '2-digit'
      })
      currentDate.value = now.toLocaleDateString('en-US', { 
        day: 'numeric',
        month: 'long'
      })
      currentYear.value = now.getFullYear().toString()
    }

    const initThreeJS = () => {
      try {
        console.log('Initializing Three.js...')
        
        // Scene setup
        scene = new THREE.Scene()
        scene.background = null // Make background transparent

        // Camera setup
        camera = new THREE.PerspectiveCamera(
          75,
          window.innerWidth / window.innerHeight,
          0.1,
          1000
        )
        camera.position.z = 15

        // Renderer setup
        renderer = new THREE.WebGLRenderer({ 
          antialias: true,
          alpha: true // Enable alpha channel for transparency
        })
        renderer.setSize(window.innerWidth, window.innerHeight)
        renderer.shadowMap.enabled = true
        renderer.shadowMap.type = THREE.PCFSoftShadowMap
        
        // Check if container exists
        if (!clockContainer.value) {
          console.error('Clock container not found!')
          return
        }
        
        clockContainer.value.appendChild(renderer.domElement)
        console.log('Renderer added to container')

        // Lighting
        const ambientLight = new THREE.AmbientLight(0x404040, 0.6)
        scene.add(ambientLight)

        const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8)
        directionalLight.position.set(10, 10, 5)
        directionalLight.castShadow = true
        scene.add(directionalLight)

        // Add point light for better 3D effect
        const pointLight = new THREE.PointLight(0xffffff, 1, 100)
        pointLight.position.set(0, 0, 10)
        scene.add(pointLight)

        console.log('Creating clock text...')
        createClockText()
        console.log('Creating stars...')
        createBubbles()
        console.log('Starting animation...')
        animate()
        
        console.log('Three.js initialization complete')
        threeJSReady.value = true
      } catch (error) {
        console.error('Error in initThreeJS:', error)
      }
    }

    const createBubbles = () => {
      // Create multiple static white stars (more visible)
      for (let i = 0; i < 25; i++) {
        const size = 0.15 + Math.random() * 0.25
        
        // Create star texture
        const canvas = document.createElement('canvas')
        const context = canvas.getContext('2d')
        canvas.width = 64
        canvas.height = 64
        
        // Clear canvas
        context.clearRect(0, 0, 64, 64)
        
        // Draw white star (more visible than black)
        context.fillStyle = '#ffffff'
        context.globalAlpha = 0.7 + Math.random() * 0.3
        
        // Create star shape
        const centerX = 32
        const centerY = 32
        const outerRadius = 18
        const innerRadius = 6
        const spikes = 5
        
        context.beginPath()
        for (let j = 0; j < spikes * 2; j++) {
          const radius = j % 2 === 0 ? outerRadius : innerRadius
          const angle = (j * Math.PI) / spikes
          const x = centerX + Math.cos(angle) * radius
          const y = centerY + Math.sin(angle) * radius
          
          if (j === 0) {
            context.moveTo(x, y)
          } else {
            context.lineTo(x, y)
          }
        }
        context.closePath()
        context.fill()
        
        const texture = new THREE.CanvasTexture(canvas)
        const material = new THREE.MeshBasicMaterial({
          map: texture,
          transparent: true
        })
        
        const geometry = new THREE.PlaneGeometry(size, size)
        const star = new THREE.Mesh(geometry, material)
        
        // Distribute stars across a wider area
        star.position.set(
          (Math.random() - 0.5) * 30,
          (Math.random() - 0.5) * 20,
          Math.random() * 5
        )
        
        // Store original position for movement
        star.originalX = star.position.x
        star.originalY = star.position.y
        star.originalZ = star.position.z
        star.speed = 0.001 + Math.random() * 0.002 // Very slow movement speed
        
        // Add to bubbles array (keeping same variable name for compatibility)
        bubbles.push(star)
        scene.add(star)
      }
    }

    const createClockText = () => {
      try {
        const time = getCurrentTime()
        const date = getCurrentDate()
        const year = getCurrentYear()
        
        // Create canvas for text with higher resolution
        const canvas = document.createElement('canvas')
        const context = canvas.getContext('2d')
        canvas.width = 1024
        canvas.height = 512
        
        // Clear canvas
        context.clearRect(0, 0, 1024, 512)
        
        // Enable text rendering optimizations
        context.imageSmoothingEnabled = false
        
        // Draw date (top) - smaller font
        context.fillStyle = '#ffffff'
        context.font = 'bold 48px Arial, sans-serif'
        context.textAlign = 'center'
        context.textBaseline = 'middle'
        context.fillText(date, 512, 128)
        
        // Draw time (middle) - largest font
        context.font = 'bold 144px Arial, sans-serif'
        context.fillText(time, 512, 256)
        
        // Draw year (bottom) - smaller font
        context.font = 'bold 48px Arial, sans-serif'
        context.fillText(year, 512, 384)
        
        const texture = new THREE.CanvasTexture(canvas)
        texture.minFilter = THREE.LinearFilter
        texture.magFilter = THREE.LinearFilter
        texture.generateMipmaps = false
        
        const material = new THREE.MeshBasicMaterial({ 
          map: texture,
          transparent: true
        })
        
        const geometry = new THREE.PlaneGeometry(12, 6)
        clockText = new THREE.Mesh(geometry, material)
        clockText.position.z = 0
        
        scene.add(clockText)
        console.log('Clock text created successfully:', { time, date, year })
      } catch (error) {
        console.error('Error creating clock text:', error)
      }
    }

    const getCurrentTime = () => {
      const now = new Date()
      const hours = now.getHours().toString().padStart(2, '0')
      const minutes = now.getMinutes().toString().padStart(2, '0')
      const seconds = now.getSeconds().toString().padStart(2, '0')
      return `${hours}:${minutes}:${seconds}`
    }

    const getCurrentDate = () => {
      const now = new Date()
      const day = now.getDate()
      const month = now.toLocaleString('en-US', { month: 'long' })
      return `${day} ${month}`
    }

    const getCurrentYear = () => {
      const now = new Date()
      return now.getFullYear().toString()
    }

    const updateClockText = () => {
      try {
        const time = getCurrentTime()
        const date = getCurrentDate()
        const year = getCurrentYear()
        
        // Create new canvas with updated text
        const canvas = document.createElement('canvas')
        const context = canvas.getContext('2d')
        canvas.width = 1024
        canvas.height = 512
        
        // Clear canvas
        context.clearRect(0, 0, 1024, 512)
        
        // Enable text rendering optimizations
        context.imageSmoothingEnabled = false
        
        // Draw date (top) - smaller font
        context.fillStyle = '#ffffff'
        context.font = 'bold 48px Arial, sans-serif'
        context.textAlign = 'center'
        context.textBaseline = 'middle'
        context.fillText(date, 512, 128)
        
        // Draw time (middle) - largest font
        context.font = 'bold 144px Arial, sans-serif'
        context.fillText(time, 512, 256)
        
        // Draw year (bottom) - smaller font
        context.font = 'bold 48px Arial, sans-serif'
        context.fillText(year, 512, 384)
        
        // Update texture
        if (clockText && clockText.material && clockText.material.map) {
          clockText.material.map.image = canvas
          clockText.material.map.needsUpdate = true
        }
      } catch (error) {
        console.error('Error updating clock text:', error)
      }
    }

    const handleMouseMove = (event) => {
      // Mouse interaction removed - stars are static
    }

    const updateBubbles = () => {
      // Move stars slowly to the right
      bubbles.forEach((star) => {
        // Move star to the right
        star.position.x += star.speed
        
        // Wrap around when star goes off the right edge
        if (star.position.x > 15) {
          star.position.x = -15
        }
      })
    }

    const animate = () => {
      animationId = requestAnimationFrame(animate)
      
      // Update stars movement
      updateBubbles()
      
      // Gentle floating animation for the text only
      if (clockText) {
        clockText.position.y = Math.sin(Date.now() * 0.001) * 0.5
        clockText.rotation.z = Math.sin(Date.now() * 0.0005) * 0.02
      }
      
      renderer.render(scene, camera)
    }

    const handleResize = () => {
      camera.aspect = window.innerWidth / window.innerHeight
      camera.updateProjectionMatrix()
      renderer.setSize(window.innerWidth, window.innerHeight)
    }

    onMounted(() => {
      console.log('Component mounted, starting initialization...')
      console.log('Clock container:', clockContainer.value)
      
      // Initialize fallback time
      updateFallbackTime()
      setInterval(updateFallbackTime, 1000)
      
      // Wait a bit for DOM to be ready
      setTimeout(() => {
        try {
          initThreeJS()
          updateClockText()
          updateGradient()
          setInterval(updateClockText, 1000)
          window.addEventListener('resize', handleResize)
          console.log('Initialization complete')
        } catch (error) {
          console.error('Error in onMounted:', error)
        }
      }, 100)
    })

    onUnmounted(() => {
      if (animationId) {
        cancelAnimationFrame(animationId)
      }
      window.removeEventListener('resize', handleResize)
      if (renderer) {
        renderer.dispose()
      }
    })

    return {
      clockContainer,
      color1,
      color2,
      showColorPicker,
      threeJSReady,
      currentTime,
      currentDate,
      currentYear,
      toggleColorPicker,
      updateGradient,
      handleMouseMove
    }
  }
}
</script>

<style scoped>
.clock-canvas {
  width: 100%;
  height: 100%;
}

.color-picker-container {
  position: absolute;
  top: 20px;
  right: 20px;
  z-index: 10;
}

.color-picker-toggle {
  width: 40px;
  height: 40px;
  border: none;
  border-radius: 8px;
  background: rgba(0, 0, 0, 0.3);
  backdrop-filter: blur(10px);
  color: white;
  font-size: 16px;
  cursor: pointer;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  justify-content: center;
}

.color-picker-toggle:hover {
  background: rgba(0, 0, 0, 0.5);
  transform: scale(1.05);
}

.color-picker {
  position: absolute;
  top: 50px;
  right: 0;
  background: rgba(0, 0, 0, 0.3);
  padding: 10px;
  border-radius: 8px;
  backdrop-filter: blur(10px);
  display: flex;
  flex-direction: column;
  gap: 8px;
  animation: slideDown 0.3s ease;
}

@keyframes slideDown {
  from {
    opacity: 0;
    transform: translateY(-10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.color-input {
  width: 40px;
  height: 40px;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  background: none;
}

.color-input::-webkit-color-swatch-wrapper {
  padding: 0;
}

.color-input::-webkit-color-swatch {
  border: none;
  border-radius: 8px;
}

.watermark {
  position: absolute;
  bottom: 15px;
  right: 15px;
  color: rgba(255, 255, 255, 0.4);
  font-size: 12px;
  font-family: 'Arial', sans-serif;
  font-weight: 300;
  z-index: 5;
  pointer-events: none;
  user-select: none;
}

.fallback-clock {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  text-align: center;
  color: white;
  z-index: 10;
  background: rgba(0, 0, 0, 0.3);
  padding: 30px 40px;
  border-radius: 15px;
  backdrop-filter: blur(10px);
}

.fallback-date {
  font-size: 24px;
  font-weight: 400;
  margin-bottom: 10px;
}

.fallback-time {
  font-size: 48px;
  font-weight: 600;
  margin-bottom: 10px;
  letter-spacing: 2px;
}

.fallback-year {
  font-size: 24px;
  font-weight: 400;
}
</style> 