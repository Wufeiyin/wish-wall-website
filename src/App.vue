<template>
  <div class="container">
    <!-- 顶部标题区 -->
    <header class="header">
      <h1 class="title">🎉 山大小青马愿望共享墙 🎉</h1>
      <p class="subtitle">写下你的愿望，与所有人分享祝福</p>
    </header>

    <!-- 实时更新提示 -->
    <div v-if="newWishCount > 0" class="update-notice" @click="loadWishes">
      ✨ 有 {{ newWishCount }} 条新愿望，点击查看
    </div>

    <!-- 愿望输入区 -->
    <section class="input-section">
      <div class="input-box">
        <textarea
          v-model="newWish"
          @input="onInput"
          placeholder="许下你的新年愿望吧（最多500字）..."
          maxlength="500"
          class="wish-input"
          :disabled="isSubmitting"
        ></textarea>
        <div class="input-footer">
          <span class="word-count">{{ newWish.length }}/500</span>
          <button
            @click="submitWish"
            :class="['submit-btn', { active: newWish.length > 0 }]"
            :disabled="isSubmitting || !newWish.trim()"
          >
            {{ isSubmitting ? '发布中...' : '分享愿望' }}
          </button>
        </div>
      </div>
    </section>

    <!-- 愿望展示墙 -->
    <section class="wall-section">
      <!-- 排序切换与刷新区域 -->
      <div class="section-header">
        <div class="sort-buttons">
          <button
            :class="['sort-btn', { active: sortType === 'latest' }]"
            @click="switchSortType('latest')"
          >
            <span class="btn-icon">🕒</span>
            <span class="btn-text">最新</span>
          </button>
          <button
            :class="['sort-btn', { active: sortType === 'hottest' }]"
            @click="switchSortType('hottest')"
          >
            <span class="btn-icon">🔥</span>
            <span class="btn-text">最热</span>
          </button>
        </div>
        <div class="header-right">
          <span v-if="lastUpdateTime" class="update-time">{{ lastUpdateTime }}</span>
          <button class="refresh-btn" @click="loadWishes">
            <span class="refresh-icon">🔄</span>
            <span class="refresh-text">刷新</span>
          </button>
        </div>
      </div>

      <!-- 排序说明 -->
      <div class="sort-notice">
        <span class="notice-icon">{{ sortType === 'latest' ? '🕒' : '🔥' }}</span>
        <span>{{ sortType === 'latest' ? '按发布时间排序' : '按点赞数排序' }}</span>
      </div>

      <!-- 愿望卡片网格 -->
      <div class="wish-grid">
        <div
          v-for="(wish, index) in wishes"
          :key="wish.id"
          :class="['wish-card', `card-${index % 4}`]"
        >
          <div class="card-content">
            <div v-if="wish.isTemp" class="temp-tag">提交中...</div>
            <div class="wish-text">{{ wish.content }}</div>
            <div class="card-footer">
              <div class="time-info">
                <span class="time">{{ formatTime(wish.created_at) }}</span>
                <span v-if="wish.user_info?.nickName === '我'" class="my-tag">我的</span>
              </div>
              <button
                :class="['like-btn', { liking: wish.isLiking }]"
                @click="toggleLike(wish, index)"
              >
                <span class="like-icon">{{ wish.isLiked ? '❤️' : '🤍' }}</span>
                <span class="like-count">{{ wish.like_count || 0 }}</span>
              </button>
            </div>
          </div>
        </div>
      </div>

      <!-- 空状态 -->
      <div v-if="wishes.length === 0 && !isLoading" class="empty-state">
        <!-- 可以放一个默认图片或图标 -->
        <div class="empty-icon">📝</div>
        <p class="empty-text">还没有愿望，快来第一个分享吧！</p>
        <p class="empty-hint">点击“分享愿望”发布第一条愿望</p>
      </div>
    </section>
  </div>
</template>

<script>
import { ref, onMounted, onUnmounted } from 'vue'
import { createClient } from '@supabase/supabase-js'

// 🔑 ⚠️ 重要：替换成你自己的 Supabase 项目信息！
const supabaseUrl = 'https://jujamrafpobzkmcnzcpz.supabase.co'
const supabaseAnonKey = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6Imp1amFtcmFmcG9iemttY256Y3B6Iiwicm9sZSI6ImFub24iLCJpYXQiOjE3Njk5Nzc3MDMsImV4cCI6MjA4NTU1MzcwM30.OfY0J6G9_Z5w9HI_sJL5quITHRKmIuPm-1WDeBE3-Tw'
const supabase = createClient(supabaseUrl, supabaseAnonKey)

export default {
  name: 'App',
  setup() {
    // 状态数据
    const newWish = ref('')
    const wishes = ref([])
    const isLoading = ref(false)
    const isSubmitting = ref(false)
    const sortType = ref('latest')
    const lastUpdateTime = ref('')
    const newWishCount = ref(0)

    // 格式化时间函数
    const formatTime = (dateString, showSeconds = false) => {
      if (!dateString) return ''
      const d = new Date(dateString)
      const now = new Date()
      const diff = now - d
      if (diff < 60000) return '刚刚'
      if (diff < 3600000) return `${Math.floor(diff / 60000)}分钟前`
      if (d.toDateString() === now.toDateString()) {
        return showSeconds 
          ? `${d.getHours().toString().padStart(2, '0')}:${d.getMinutes().toString().padStart(2, '0')}`
          : '今天'
      }
      if (d.getFullYear() === now.getFullYear()) {
        return `${d.getMonth() + 1}月${d.getDate()}日`
      }
      return `${d.getFullYear()}年${d.getMonth() + 1}月`
    }

    // 加载愿望列表
    const loadWishes = async () => {
      isLoading.value = true
      try {
        // 构建查询，根据排序类型选择字段
        let query = supabase
          .from('wishes')
          .select('*')
        
        if (sortType.value === 'latest') {
          query = query.order('created_at', { ascending: false })
        } else {
          query = query.order('like_count', { ascending: false })
        }

        const { data, error } = await query
        if (error) throw error

        wishes.value = data.map(item => ({
          ...item,
          isLiked: false, // 可以从 localStorage 获取真实状态
          isLiking: false
        }))

        lastUpdateTime.value = formatTime(new Date(), true)
        newWishCount.value = 0 // 加载后清空新愿望计数
      } catch (error) {
        console.error('加载失败:', error)
      } finally {
        isLoading.value = false
      }
    }

    // 提交愿望
    const submitWish = async () => {
      const content = newWish.value.trim()
      if (!content || content.length > 500) return

      isSubmitting.value = true
      
      // 创建临时愿望（立即显示优化体验）
      const tempWish = {
        id: 'temp_' + Date.now(),
        content: content,
        created_at: new Date().toISOString(),
        like_count: 0,
        user_info: { nickName: '我' },
        isTemp: true
      }

      // 立即显示在列表顶部（如果是“最新”排序）
      if (sortType.value === 'latest') {
        wishes.value = [tempWish, ...wishes.value]
      } else {
        wishes.value = [...wishes.value, tempWish] // 最热排序放末尾
      }
      newWish.value = ''

      try {
        const { data, error } = await supabase
          .from('wishes')
          .insert([
            { 
              content: content,
              like_count: 0,
              user_info: { nickName: '匿名用户' }
            }
          ])
          .select()
          .single()

        if (error) throw error

        // 用真实数据替换临时数据
        const index = wishes.value.findIndex(w => w.id === tempWish.id)
        if (index !== -1) {
          wishes.value[index] = { ...data, isLiked: false, isLiking: false }
        }

        // 如果是“最热”排序，需要重新加载以确保顺序正确
        if (sortType.value === 'hottest') {
          setTimeout(loadWishes, 500)
        }

      } catch (error) {
        console.error('发布失败:', error)
        // 移除临时愿望
        wishes.value = wishes.value.filter(w => w.id !== tempWish.id)
      } finally {
        isSubmitting.value = false
      }
    }

    // 点赞/取消点赞
    const toggleLike = async (wish, index) => {
      if (wish.isLiking) return

      const newIsLiked = !wish.isLiked
      const newLikeCount = newIsLiked ? (wish.like_count || 0) + 1 : Math.max(0, (wish.like_count || 0) - 1)

      // 立即更新UI
      wish.isLiked = newIsLiked
      wish.like_count = newLikeCount
      wish.isLiking = true

      try {
        const { error } = await supabase
          .from('wishes')
          .update({ like_count: newLikeCount })
          .eq('id', wish.id)

        if (error) throw error

        // 如果是最热排序，点赞后需要重新加载
        if (sortType.value === 'hottest') {
          setTimeout(loadWishes, 300)
        }
      } catch (error) {
        console.error('点赞失败:', error)
        // 恢复状态
        wish.isLiked = !newIsLiked
        wish.like_count = wish.like_count || 0
      } finally {
        wish.isLiking = false
      }
    }

    // 切换排序类型
    const switchSortType = (type) => {
      if (sortType.value === type) return
      sortType.value = type
      loadWishes()
    }

    // 开启实时监听（Supabase 的实时功能）
    const setupRealtime = () => {
      const subscription = supabase
        .channel('wishes-changes')
        .on('postgres_changes', 
          { event: 'INSERT', schema: 'public', table: 'wishes' }, 
          (payload) => {
            console.log('有新愿望！', payload)
            newWishCount.value += 1
          }
        )
        .subscribe()

      return subscription
    }

    // 生命周期
    onMounted(() => {
      loadWishes()
      const subscription = setupRealtime()
      
      // 定期刷新
      const intervalId = setInterval(loadWishes, 60000) // 每60秒刷新一次
      
      onUnmounted(() => {
        subscription.unsubscribe()
        clearInterval(intervalId)
      })
    })

    return {
      newWish,
      wishes,
      isLoading,
      isSubmitting,
      sortType,
      lastUpdateTime,
      newWishCount,
      onInput: (e) => { newWish.value = e.target.value },
      submitWish,
      toggleLike,
      switchSortType,
      loadWishes,
      formatTime
    }
  }
}
</script>

<style>
/* 基础样式 */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Helvetica Neue', sans-serif;
  background: #f5f7fa;
}

.container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  min-height: 100vh;
}

/* 标题区 */
.header {
  text-align: center;
  padding: 40px 20px;
  background: linear-gradient(to right, #D32F2F, #F57C00);
  margin: -20px -20px 30px;
  border-radius: 0 0 25px 25px;
  color: white;
  box-shadow: 0 6px 20px rgba(213, 47, 47, 0.25);
}

.title {
  font-size: 2.2em;
  font-weight: bold;
  margin-bottom: 10px;
  text-shadow: 0 2px 4px rgba(0,0,0,0.2);
}

.subtitle {
  font-size: 1.1em;
  opacity: 0.9;
}

/* 实时更新提示 */
.update-notice {
  background: linear-gradient(to right, #4CAF50, #8BC34A);
  color: white;
  padding: 15px 25px;
  margin: 0 20px 25px;
  border-radius: 15px;
  text-align: center;
  font-size: 1.1em;
  font-weight: 500;
  cursor: pointer;
  box-shadow: 0 4px 15px rgba(76, 175, 80, 0.25);
  transition: transform 0.2s;
}

.update-notice:hover {
  transform: translateY(-2px);
}

/* 输入区域 */
.input-section {
  padding: 0 20px;
  margin-bottom: 35px;
}

.input-box {
  background: white;
  border-radius: 20px;
  padding: 30px;
  box-shadow: 0 8px 25px rgba(213, 47, 47, 0.15);
  border: 1px solid #FFE0B2;
}

.wish-input {
  width: 100%;
  min-height: 120px;
  font-size: 1.1em;
  line-height: 1.5;
  color: #5D4037;
  border-radius: 12px;
  padding: 18px;
  background: #f8f9fa;
  border: 2px solid #e0e0e0;
  resize: vertical;
  transition: border-color 0.3s;
  font-family: inherit;
}

.wish-input:focus {
  outline: none;
  border-color: #D32F2F;
  background: white;
}

.input-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 25px;
}

.word-count {
  color: #999;
  font-size: 0.95em;
}

.submit-btn {
  background: linear-gradient(to right, #757575, #9E9E9E);
  color: white;
  border-radius: 50px;
  padding: 0 50px;
  height: 55px;
  border: none;
  font-size: 1.1em;
  font-weight: 500;
  cursor: not-allowed;
  transition: all 0.3s;
  opacity: 0.8;
}

.submit-btn.active {
  background: linear-gradient(to right, #D32F2F, #F57C00);
  opacity: 1;
  cursor: pointer;
  box-shadow: 0 6px 20px rgba(213, 47, 47, 0.3);
}

.submit-btn.active:hover {
  opacity: 0.95;
  transform: translateY(-2px);
}

/* 排序功能区 */
.section-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 25px;
  padding: 0 20px;
  flex-wrap: wrap;
  gap: 15px;
}

.sort-buttons {
  display: flex;
  gap: 12px;
  background: #f8f9fa;
  border-radius: 40px;
  padding: 6px;
  border: 2px solid #FFE0B2;
}

.sort-btn {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 14px 26px;
  border-radius: 35px;
  border: none;
  background: transparent;
  font-size: 1em;
  color: #666;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.3s;
  min-width: 120px;
  justify-content: center;
}

.sort-btn.active {
  background: linear-gradient(135deg, #D32F2F, #F57C00);
  color: white;
  font-weight: bold;
  box-shadow: 0 4px 12px rgba(213, 47, 47, 0.25);
}

.sort-btn:hover:not(.active) {
  background: #f0f0f0;
}

.header-right {
  display: flex;
  align-items: center;
  gap: 15px;
}

.update-time {
  font-size: 0.9em;
  color: #999;
}

.refresh-btn {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 12px 22px;
  background: white;
  border-radius: 35px;
  border: 2px solid #FFCCBC;
  font-size: 0.95em;
  color: #D32F2F;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.3s;
}

.refresh-btn:hover {
  background: #FFF3E0;
  transform: rotate(15deg);
}

/* 愿望卡片网格 */
.wish-grid {
  display: grid;
  grid-template-columns: 1fr;
  gap: 25px;
  margin-bottom: 40px;
  padding: 0 20px;
}

.wish-card {
  background: white;
  border-radius: 18px;
  overflow: hidden;
  box-shadow: 0 6px 18px rgba(0, 0, 0, 0.08);
  transition: all 0.3s;
  position: relative;
  border-top: 6px solid;
}

.wish-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 12px 28px rgba(0, 0, 0, 0.12);
}

/* 卡片颜色 */
.wish-card.card-0 { border-top-color: #D32F2F; }
.wish-card.card-1 { border-top-color: #F57C00; }
.wish-card.card-2 { border-top-color: #388E3C; }
.wish-card.card-3 { border-top-color: #1976D2; }

.card-content {
  padding: 25px;
}

.temp-tag {
  position: absolute;
  top: 12px;
  right: 12px;
  background: #FF9800;
  color: white;
  font-size: 0.8em;
  padding: 4px 10px;
  border-radius: 10px;
  z-index: 10;
}

.wish-text {
  font-size: 1.15em;
  line-height: 1.5;
  color: #5D4037;
  margin-bottom: 20px;
  min-height: 80px;
  word-break: break-word;
}

.card-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-top: 18px;
  border-top: 1px dashed #eee;
  font-size: 0.9em;
  color: #666;
}

.time-info {
  display: flex;
  align-items: center;
  gap: 10px;
}

.my-tag {
  background: #E3F2FD;
  color: #1976D2;
  padding: 3px 8px;
  border-radius: 6px;
  font-size: 0.85em;
}

.like-btn {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 20px;
  background: #FFF3E0;
  border-radius: 30px;
  border: 2px solid #FFB74D;
  font-size: 1em;
  font-weight: bold;
  color: #F57C00;
  cursor: pointer;
  transition: all 0.3s;
}

.like-btn:hover {
  background: #FFE0B2;
  transform: scale(1.05);
}

/* 空状态 */
.empty-state {
  text-align: center;
  padding: 60px 20px;
  background: white;
  border-radius: 20px;
  margin: 30px 20px;
  box-shadow: 0 6px 18px rgba(0, 0, 0, 0.05);
}

.empty-icon {
  font-size: 4em;
  margin-bottom: 20px;
  opacity: 0.6;
}

.empty-text {
  font-size: 1.4em;
  color: #999;
  margin-bottom: 10px;
  font-weight: 500;
}

.empty-hint {
  color: #ccc;
  font-size: 1em;
}

/* 加载状态 */
.loading-state {
  text-align: center;
  padding: 60px 20px;
  color: #999;
}
</style>