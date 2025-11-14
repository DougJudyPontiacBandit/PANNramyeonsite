<template>
  <div class="promotions-page">
    <!-- Page Header -->
    <div class="page-header">
      <h1 class="page-title">🎁 Active Promotions</h1>
      <p class="page-subtitle">Discover amazing deals and save on your favorite items!</p>
    </div>
    
    <!-- Loading State -->
    <div v-if="isLoading" class="loading-container">
      <div class="loading-spinner-big"></div>
      <p>Loading promotions...</p>
    </div>
    
    <!-- Error State -->
    <div v-else-if="error" class="error-container">
      <div class="error-icon">⚠️</div>
      <h3>Failed to Load Promotions</h3>
      <p>{{ error }}</p>
      <button @click="fetchActivePromotions" class="retry-btn">Retry</button>
    </div>
    
    <!-- No Promotions State -->
    <div v-else-if="!isLoading && promotions.length === 0" class="empty-promotions">
      <div class="empty-icon">🎁</div>
      <h3>No Active Promotions</h3>
      <p>Check back later for exciting deals and offers!</p>
    </div>
    
    <!-- Main Content - Database Promotions Only -->
    <main v-else class="main-content">
      <!-- Promotions Grid -->
      <section class="promotions-grid">
        <div 
          v-for="promotion in promotions" 
          :key="promotion.promotion_id"
          class="promo-card"
          :class="{ 'disabled': !isLoggedIn, 'featured': isFeaturedPromotion(promotion) }"
          @click="handlePromotionClick(promotion.promotion_id, promotion.name)"
        >
          <!-- Promotion Badge -->
          <div class="promo-badge" :class="getPromotionTypeClass(promotion.type)">
            {{ getPromotionTypeLabel(promotion.type) }}
          </div>
          
          <!-- Promotion Image -->
          <div class="promo-image">
            <img :src="getPromotionImage(promotion)" :alt="promotion.name" />
            <div class="promo-overlay">
              <div class="promo-discount">
                {{ getDiscountDisplay(promotion) }}
              </div>
            </div>
          </div>
          
          <!-- Promotion Content -->
          <div class="promo-content">
            <h3 class="promo-title">{{ promotion.name }}</h3>
            <p class="promo-description">{{ promotion.description || 'Amazing deal awaits!' }}</p>
            
            <!-- Promotion Details -->
            <div class="promo-details">
              <div class="detail-item">
                <span class="detail-icon">📅</span>
                <span class="detail-text">Valid until {{ formatDate(promotion.end_date) }}</span>
              </div>
              <div v-if="promotion.usage_limit" class="detail-item">
                <span class="detail-icon">🎯</span>
                <span class="detail-text">{{ getRemainingUsage(promotion) }} left</span>
              </div>
            </div>
            
            <!-- Action Buttons -->
            <div class="promo-actions">
              <button class="use-btn" @click.stop="handlePromotionClick(promotion.promotion_id, promotion.name)">
                <span class="btn-icon">🎫</span>
                <span>Use Now</span>
              </button>
              <button 
                v-if="isLoggedIn" 
                class="save-btn" 
                @click.stop="savePromotion(promotion.promotion_id, promotion.name, getDiscountDisplay(promotion))" 
                :disabled="isSaving(promotion.promotion_id)"
              >
                <span v-if="!isSaving(promotion.promotion_id)" class="btn-icon">💾</span>
                <span v-else class="loading-spinner-small">⌛</span>
                <span v-if="!isSaving(promotion.promotion_id)">Save</span>
                <span v-else>Saving...</span>
              </button>
            </div>
          </div>
        </div>
      </section>
    </main>

    <!-- Modal for Promotion Code -->
    <div v-if="showModal" class="modal-overlay" @click="closeModal">
      <div class="modal-content" @click.stop>
        <h3>{{ modalTitle }}</h3>
        <div class="text-code">
          <p class="code-label">Your Promotion Code:</p>
          <div class="code-display">{{ currentCode }}</div>
          <p class="code-instruction">Show this code to the cashier or enter it at checkout</p>
        </div>
        <button @click="closeModal" class="close-btn">Close</button>
      </div>
    </div>
  </div>
</template>

<script>
import { usePromotions } from '../composables/api/usePromotions.js'

export default {
  name: 'Promotions',
  emits: ['setCurrentPage'],
  props: {
    isLoggedIn: Boolean
  },
  setup() {
    // Initialize promotions composable
    const promotions = usePromotions();
    
    return {
      // Expose composable methods and state
      ...promotions
    };
  },
  data() {
    return {
      showModal: false,
      modalTitle: '',
      currentCode: '',
      savingPromotions: {}, // Track which promotions are being saved
      ramyeonHero: require('@/assets/food/ramyeon-hero.jpg'),
      // activePromotions now comes from usePromotions composable
      // loading and error now come from usePromotions composable
      // Fallback images for promotions
      fallbackImages: {
        'ramyeon': require('@/assets/food/ramyeon-hero.jpg'),
        'kimchi': require('@/assets/food/kimchi.jpg'),
        'bulgogi': require('@/assets/food/bulgogi.jpg'),
        'corndog': require('@/assets/food/corn-dog.jpg'),
        'fishcake': require('@/assets/food/fish-cake.jpg'),
        'tteokbokki': require('@/assets/food/tteokbokki.jpg'),
      }
    }
  },
  computed: {
    // No computed properties needed - display all promotions directly
  },
  async mounted() {
    // Initialize promotions when component is mounted
    await this.fetchActivePromotions();
  },
  methods: {
    async fetchActivePromotions() {
      try {
        console.log('🎯 Fetching active promotions using composable...')
        await this.getActivePromotions()
        console.log('✅ Loaded', this.promotions.length, 'active promotions')
        console.log('Promotions:', this.promotions)
      } catch (error) {
        console.error('❌ Error fetching promotions:', error)
        this.showErrorMessage('Could not load promotions. Please try again later.')
      }
    },
    
    getDiscountDisplay(promotion) {
      // Generate discount display based on promotion type
      if (promotion.type === 'percentage') {
        return `${promotion.discount_value}% OFF`
      } else if (promotion.type === 'fixed_amount') {
        return `₱${promotion.discount_value} OFF`
      } else if (promotion.type === 'buy_x_get_y') {
        const config = promotion.discount_config || {}
        return `Buy ${config.buy_quantity || 2} Get ${config.get_quantity || 1} Free`
      }
      return 'Special Offer'
    },
    
    getPromotionTypeLabel(type) {
      const labels = {
        'percentage': '% DISCOUNT',
        'fixed_amount': 'FIXED DISCOUNT',
        'buy_x_get_y': 'BUY & GET',
        'free_shipping': 'FREE SHIPPING'
      }
      return labels[type] || 'SPECIAL OFFER'
    },
    
    getPromotionTypeClass(type) {
      return `badge-${type.replace('_', '-')}`
    },
    
    isFeaturedPromotion(promotion) {
      // Mark promotions with high discount values as featured
      if (promotion.type === 'percentage' && promotion.discount_value >= 30) {
        return true
      }
      if (promotion.type === 'fixed_amount' && promotion.discount_value >= 50) {
        return true
      }
      return false
    },
    
    formatDate(dateString) {
      if (!dateString) return 'N/A'
      const date = new Date(dateString)
      return date.toLocaleDateString('en-US', { month: 'short', day: 'numeric', year: 'numeric' })
    },
    
    getRemainingUsage(promotion) {
      if (!promotion.usage_limit) return 'Unlimited'
      const remaining = promotion.usage_limit - (promotion.current_usage || 0)
      return remaining > 0 ? remaining : 'Sold out'
    },
    
    getPromotionImage(promotion) {
      // Try to match promotion name to an image
      const nameLower = promotion.name.toLowerCase()
      
      if (nameLower.includes('ramyeon') || nameLower.includes('ramen')) {
        return this.fallbackImages.ramyeon
      } else if (nameLower.includes('kimchi')) {
        return this.fallbackImages.kimchi
      } else if (nameLower.includes('bulgogi')) {
        return this.fallbackImages.bulgogi
      } else if (nameLower.includes('corn') || nameLower.includes('dog')) {
        return this.fallbackImages.corndog
      } else if (nameLower.includes('fish') || nameLower.includes('cake')) {
        return this.fallbackImages.fishcake
      } else if (nameLower.includes('tteok')) {
        return this.fallbackImages.tteokbokki
      }
      
      // Default to ramyeon hero image
      return this.fallbackImages.ramyeon
    },
    
    goBack() {
      this.$emit('setCurrentPage', 'Home')
    },
    
    handlePromotionClick(code, title) {
      if (!this.isLoggedIn) {
        this.showErrorMessage('Please log in to access promotions!')
        return
      }
      this.showPromoCode(code, title)
    },
    
    isSaving(code) {
      return this.savingPromotions[code] === true
    },
    showPromoCode(code, title) {
      this.currentCode = code
      this.modalTitle = title
      this.showModal = true
    },
    closeModal() {
      this.showModal = false
      this.currentCode = ''
      this.modalTitle = ''
    },

    async savePromotion(code, title, discount) {
      if (!this.isLoggedIn) {
        this.showErrorMessage('Please log in to save promotions!')
        return
      }

      // Check if already saving this promotion
      if (this.savingPromotions[code]) {
        return
      }

      // Set saving state for this specific promotion
      this.savingPromotions[code] = true

      try {
        // Check if already saved
        const savedVouchers = JSON.parse(localStorage.getItem('ramyeon_saved_vouchers') || '[]')
        const exists = savedVouchers.find(v => v.code === code)
        
        if (exists) {
          this.showErrorMessage('This promotion is already saved!')
          return
        }

        // Find the full promotion data from active promotions
        let fullPromotion = this.activePromotions.find(p => p.promotion_id === code)
        
        if (!fullPromotion) {
          // Fallback: try to fetch from API
          console.log('Promotion not found in active list, fetching from API...')
          try {
            const response = await this.getPromotion(code)
            
            if (response.success && response.promotion) {
              fullPromotion = response.promotion
            } else {
              throw new Error('Promotion not found')
            }
          } catch (apiError) {
            console.error('Failed to fetch promotion from API:', apiError)
            throw new Error('Promotion not found')
          }
        }

        // Create promotion voucher object with full data
        const promotionVoucher = {
          id: Date.now(), // Generate unique ID for local storage
          promotion_id: fullPromotion.promotion_id,
          title: fullPromotion.name,
          subtitle: fullPromotion.description || 'Promotion Offer',
          discount: discount,
          code: code,
          type: 'promotion',
          // Store full promotion data for later use
          promotionData: {
            type: fullPromotion.type,
            discount_value: fullPromotion.discount_value,
            target_type: fullPromotion.target_type,
            target_ids: fullPromotion.target_ids,
            start_date: fullPromotion.start_date,
            end_date: fullPromotion.end_date,
            usage_limit: fullPromotion.usage_limit,
            current_usage: fullPromotion.current_usage
          },
          qrCode: `${code}-QR-${Date.now()}`,
          savedAt: new Date().toISOString()
        }

        // Save to localStorage
        savedVouchers.push(promotionVoucher)
        localStorage.setItem('ramyeon_saved_vouchers', JSON.stringify(savedVouchers))

        console.log('✅ Promotion saved:', promotionVoucher)

        // Show success message
        this.showSuccessMessage('Promotion saved! Redirecting to profile...')

        // Redirect to profile after short delay
        setTimeout(() => {
          this.$emit('setCurrentPage', 'Profile')
        }, 1000)

      } catch (error) {
        console.error('Error saving promotion:', error)
        this.showErrorMessage('Failed to save promotion. Please try again.')
      } finally {
        // Clear saving state for this specific promotion
        this.savingPromotions[code] = false
      }
    },

    showSuccessMessage(message) {
      // Create success notification
      const notification = document.createElement('div')
      notification.innerHTML = `
        <div style="
          position: fixed;
          top: 20px;
          right: 20px;
          background: linear-gradient(135deg, #28a745, #20c997);
          color: white;
          padding: 15px 25px;
          border-radius: 12px;
          box-shadow: 0 8px 25px rgba(40, 167, 69, 0.3);
          z-index: 9999;
          font-family: 'Poppins', sans-serif;
          font-weight: 600;
          animation: slideInRight 0.3s ease-out;
          display: flex;
          align-items: center;
          gap: 10px;
        ">
          <span style="font-size: 1.2rem;">✅</span>
          ${message}
        </div>
        <style>
          @keyframes slideInRight {
            from { transform: translateX(100%); opacity: 0; }
            to { transform: translateX(0); opacity: 1; }
          }
        </style>
      `

      document.body.appendChild(notification)

      // Remove after 3 seconds
      setTimeout(() => {
        if (notification.parentNode) {
          notification.style.animation = 'slideInRight 0.3s ease-in reverse'
          setTimeout(() => {
            document.body.removeChild(notification)
          }, 300)
        }
      }, 3000)
    },

    showErrorMessage(message) {
      // Create error notification
      const notification = document.createElement('div')
      notification.innerHTML = `
        <div style="
          position: fixed;
          top: 20px;
          right: 20px;
          background: linear-gradient(135deg, #dc3545, #c82333);
          color: white;
          padding: 15px 25px;
          border-radius: 12px;
          box-shadow: 0 8px 25px rgba(220, 53, 69, 0.3);
          z-index: 9999;
          font-family: 'Poppins', sans-serif;
          font-weight: 600;
          animation: slideInRight 0.3s ease-out;
          display: flex;
          align-items: center;
          gap: 10px;
        ">
          <span style="font-size: 1.2rem;">❌</span>
          ${message}
        </div>
        <style>
          @keyframes slideInRight {
            from { transform: translateX(100%); opacity: 0; }
            to { transform: translateX(0); opacity: 1; }
          }
        </style>
      `

      document.body.appendChild(notification)

      // Remove after 3 seconds
      setTimeout(() => {
        if (notification.parentNode) {
          notification.style.animation = 'slideInRight 0.3s ease-in reverse'
          setTimeout(() => {
            document.body.removeChild(notification)
          }, 300)
        }
      }, 3000)
    }
  }
}
</script>

<style scoped>
/* ============================================
   MODERN PROMOTIONS PAGE DESIGN
   ============================================ */

.promotions-page {
  min-height: 100vh;
  background: linear-gradient(135deg, #fef5f5 0%, #fff8f6 50%, #fef1ee 100%);
  padding: 30px 20px;
  font-family: 'Poppins', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
}

/* Page Header */
.page-header {
  text-align: center;
  margin-bottom: 50px;
  animation: fadeInDown 0.6s ease-out;
}

.page-title {
  font-size: 3rem;
  font-weight: 800;
  background: linear-gradient(135deg, #ff6f61, #ff4757, #ff3838);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  margin: 0 0 15px 0;
  letter-spacing: -1px;
}

.page-subtitle {
  font-size: 1.2rem;
  color: #666;
  font-weight: 500;
  margin: 0;
}

/* Main Content */
.main-content {
  max-width: 1400px;
  margin: 0 auto;
}

/* Promotions Grid */
.promotions-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(350px, 1fr));
  gap: 30px;
  padding: 20px 0;
}

/* Promotion Card */
.promo-card {
  background: white;
  border-radius: 24px;
  overflow: hidden;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.08);
  transition: all 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  cursor: pointer;
  position: relative;
  border: 2px solid transparent;
  animation: fadeInUp 0.6s ease-out backwards;
}

.promo-card:nth-child(1) { animation-delay: 0.1s; }
.promo-card:nth-child(2) { animation-delay: 0.2s; }
.promo-card:nth-child(3) { animation-delay: 0.3s; }
.promo-card:nth-child(4) { animation-delay: 0.4s; }
.promo-card:nth-child(5) { animation-delay: 0.5s; }
.promo-card:nth-child(6) { animation-delay: 0.6s; }

.promo-card:hover {
  transform: translateY(-12px) scale(1.02);
  box-shadow: 0 20px 60px rgba(255, 111, 97, 0.25);
  border-color: rgba(255, 111, 97, 0.3);
}

.promo-card.featured {
  border-color: #ff6f61;
  background: linear-gradient(135deg, #fff 0%, #fff8f6 100%);
}

.promo-card.featured::before {
  content: '⭐ FEATURED';
  position: absolute;
  top: 15px;
  left: 15px;
  background: linear-gradient(135deg, #ffd700, #ffed4e);
  color: #333;
  padding: 6px 14px;
  border-radius: 20px;
  font-size: 0.7rem;
  font-weight: 800;
  letter-spacing: 0.5px;
  z-index: 10;
  box-shadow: 0 4px 15px rgba(255, 215, 0, 0.4);
}

/* Promotion Badge */
.promo-badge {
  position: absolute;
  top: 15px;
  right: 15px;
  padding: 8px 16px;
  border-radius: 20px;
  font-size: 0.75rem;
  font-weight: 700;
  letter-spacing: 0.5px;
  z-index: 10;
  backdrop-filter: blur(10px);
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
}

.badge-percentage {
  background: linear-gradient(135deg, #ff6f61, #ff4757);
  color: white;
}

.badge-fixed-amount {
  background: linear-gradient(135deg, #4facfe, #00f2fe);
  color: white;
}

.badge-buy-x-get-y {
  background: linear-gradient(135deg, #43e97b, #38f9d7);
  color: white;
}

.badge-free-shipping {
  background: linear-gradient(135deg, #fa709a, #fee140);
  color: white;
}

/* Promotion Image */
.promo-image {
  position: relative;
  height: 240px;
  overflow: hidden;
  background: linear-gradient(135deg, #f5f7fa, #c3cfe2);
}

.promo-image img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  transition: transform 0.5s ease;
}

.promo-card:hover .promo-image img {
  transform: scale(1.1) rotate(2deg);
}

/* Promotion Overlay */
.promo-overlay {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  background: linear-gradient(to top, rgba(0, 0, 0, 0.8), transparent);
  padding: 30px 20px 20px;
  transform: translateY(10px);
  opacity: 0;
  transition: all 0.4s ease;
}

.promo-card:hover .promo-overlay {
  transform: translateY(0);
  opacity: 1;
}

.promo-discount {
  font-size: 2.5rem;
  font-weight: 900;
  color: white;
  text-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
  letter-spacing: -1px;
}

/* Promotion Content */
.promo-content {
  padding: 25px;
}

.promo-title {
  font-size: 1.5rem;
  font-weight: 700;
  color: #333;
  margin: 0 0 12px 0;
  line-height: 1.3;
  overflow: hidden;
  text-overflow: ellipsis;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
}

.promo-description {
  font-size: 0.95rem;
  color: #666;
  line-height: 1.6;
  margin: 0 0 20px 0;
  overflow: hidden;
  text-overflow: ellipsis;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
}

/* Promotion Details */
.promo-details {
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin-bottom: 20px;
  padding: 15px;
  background: linear-gradient(135deg, #f8f9fa, #e9ecef);
  border-radius: 12px;
}

.detail-item {
  display: flex;
  align-items: center;
  gap: 10px;
  font-size: 0.9rem;
  color: #555;
}

.detail-icon {
  font-size: 1.2rem;
  filter: drop-shadow(0 2px 4px rgba(0, 0, 0, 0.1));
}

.detail-text {
  font-weight: 500;
}

/* Promotion Actions */
.promo-actions {
  display: flex;
  gap: 12px;
}

.use-btn,
.save-btn {
  flex: 1;
  padding: 14px 20px;
  border: none;
  border-radius: 14px;
  font-size: 0.95rem;
  font-weight: 700;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.use-btn {
  background: linear-gradient(135deg, #ff6f61, #ff4757);
  color: white;
  box-shadow: 0 6px 20px rgba(255, 111, 97, 0.3);
}

.use-btn:hover {
  transform: translateY(-3px);
  box-shadow: 0 10px 30px rgba(255, 111, 97, 0.5);
}

.use-btn:active {
  transform: translateY(-1px);
}

.save-btn {
  background: white;
  color: #ff6f61;
  border: 2px solid #ff6f61;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.05);
}

.save-btn:hover {
  background: #ff6f61;
  color: white;
  transform: translateY(-3px);
  box-shadow: 0 10px 30px rgba(255, 111, 97, 0.3);
}

.save-btn:active {
  transform: translateY(-1px);
}

.save-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
  transform: none;
}

.save-btn:disabled:hover {
  background: white;
  color: #ff6f61;
  transform: none;
}

/* Loading Spinner */
.loading-spinner-small {
  animation: spin 1s linear infinite;
}

@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

/* Disabled Card */
.promo-card.disabled {
  opacity: 0.6;
  cursor: not-allowed;
  filter: grayscale(60%);
}

.promo-card.disabled:hover {
  transform: none;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.08);
}

/* ============================================
   LOADING, ERROR, AND EMPTY STATES
   ============================================ */

.loading-container,
.error-container,
.empty-promotions {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-height: 60vh;
  padding: 40px 20px;
  text-align: center;
}

.loading-spinner-big {
  width: 60px;
  height: 60px;
  border: 4px solid rgba(255, 111, 97, 0.2);
  border-top-color: #ff6f61;
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin-bottom: 20px;
}

.loading-container p {
  font-size: 1.2rem;
  color: #666;
  font-weight: 600;
}

.error-container {
  background: white;
  border-radius: 24px;
  padding: 60px 40px;
  max-width: 500px;
  margin: 40px auto;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.1);
}

.error-icon {
  font-size: 4rem;
  margin-bottom: 20px;
}

.error-container h3 {
  font-size: 1.8rem;
  color: #dc3545;
  margin-bottom: 15px;
  font-weight: 700;
}

.error-container p {
  color: #666;
  font-size: 1.1rem;
  margin-bottom: 30px;
  line-height: 1.6;
}

.retry-btn {
  background: linear-gradient(135deg, #ff6f61, #ff4a3d);
  color: white;
  border: none;
  padding: 15px 40px;
  border-radius: 14px;
  font-size: 1.1rem;
  font-weight: 700;
  cursor: pointer;
  transition: all 0.3s ease;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  box-shadow: 0 6px 20px rgba(255, 111, 97, 0.3);
}

.retry-btn:hover {
  transform: translateY(-3px);
  box-shadow: 0 10px 30px rgba(255, 111, 97, 0.5);
}

.empty-promotions {
  background: white;
  border-radius: 24px;
  padding: 60px 40px;
  max-width: 500px;
  margin: 40px auto;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.1);
}

.empty-icon {
  font-size: 5rem;
  margin-bottom: 20px;
  opacity: 0.5;
}

.empty-promotions h3 {
  font-size: 1.8rem;
  color: #333;
  margin-bottom: 15px;
  font-weight: 700;
}

.empty-promotions p {
  color: #666;
  font-size: 1.1rem;
  line-height: 1.6;
}

/* ============================================
   ANIMATIONS
   ============================================ */

@keyframes fadeInDown {
  from {
    opacity: 0;
    transform: translateY(-30px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes fadeInUp {
  from {
    opacity: 0;
    transform: translateY(30px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* ============================================
   RESPONSIVE DESIGN
   ============================================ */

@media (max-width: 768px) {
  .page-title {
    font-size: 2.2rem;
  }
  
  .page-subtitle {
    font-size: 1rem;
  }
  
  .promotions-grid {
    grid-template-columns: 1fr;
    gap: 20px;
  }
  
  .promo-actions {
    flex-direction: column;
  }
  
  .use-btn,
  .save-btn {
    width: 100%;
  }
}

@media (max-width: 480px) {
  .promotions-page {
    padding: 20px 15px;
  }
  
  .page-header {
    margin-bottom: 30px;
  }
  
  .page-title {
    font-size: 1.8rem;
  }
  
  .promo-image {
    height: 180px;
  }
  
  .promo-content {
    padding: 20px;
  }
  
  .promo-title {
    font-size: 1.3rem;
  }
  
  .promo-discount {
    font-size: 2rem;
  }
}

/* ============================================
   MODAL STYLES
   ============================================ */

.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.75);
  backdrop-filter: blur(5px);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 9999;
  animation: fadeIn 0.3s ease;
}

.modal-content {
  background: white;
  border-radius: 24px;
  padding: 40px;
  max-width: 450px;
  width: 90%;
  max-height: 90vh;
  overflow-y: auto;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
  animation: scaleIn 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  text-align: center;
}

.modal-content h3 {
  font-size: 1.8rem;
  color: #333;
  margin: 0 0 25px 0;
  font-weight: 700;
}

.text-code {
  margin: 30px 0;
}

.code-label {
  font-size: 1rem;
  color: #666;
  font-weight: 600;
  margin-bottom: 15px;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.code-display {
  background: linear-gradient(135deg, #ff6f61, #ff4757);
  padding: 30px 20px;
  border-radius: 16px;
  font-size: 2rem;
  font-weight: 800;
  color: white;
  letter-spacing: 4px;
  margin: 20px 0;
  box-shadow: 0 8px 30px rgba(255, 111, 97, 0.4);
  text-align: center;
  font-family: 'Courier New', monospace;
  word-break: break-all;
  transition: all 0.3s ease;
}

.code-display:hover {
  transform: scale(1.05);
  box-shadow: 0 12px 40px rgba(255, 111, 97, 0.6);
}

.code-instruction {
  font-size: 0.95rem;
  color: #666;
  margin-top: 15px;
  line-height: 1.6;
}

.close-btn {
  background: linear-gradient(135deg, #ff6f61, #ff4757);
  color: white;
  border: none;
  padding: 14px 40px;
  border-radius: 14px;
  font-size: 1rem;
  font-weight: 700;
  cursor: pointer;
  transition: all 0.3s ease;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  box-shadow: 0 6px 20px rgba(255, 111, 97, 0.3);
  margin-top: 20px;
}

.close-btn:hover {
  transform: translateY(-3px);
  box-shadow: 0 10px 30px rgba(255, 111, 97, 0.5);
}

@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

@keyframes scaleIn {
  from {
    opacity: 0;
    transform: scale(0.9);
  }
  to {
    opacity: 1;
    transform: scale(1);
  }
}
</style>
