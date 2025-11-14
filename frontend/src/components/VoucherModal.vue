<template>
  <div v-if="isVisible" class="modal-overlay" @click="closeModal">
    <div class="modal-container" @click.stop>
      <div class="modal-header">
        <h2>{{ voucher.title }}</h2>
        <button class="close-btn" @click="closeModal" aria-label="Close modal">
          ✕
        </button>
      </div>
      
      <div class="modal-content">
        <div class="voucher-details">
          <div class="voucher-info">
            <h3>{{ voucher.subtitle }}</h3>
            <div class="discount-badge">
              {{ voucher.discount }}
            </div>
            <p class="voucher-description">
              {{ getVoucherDescription() }}
            </p>
          </div>
          
          <div class="code-section">
            <div class="code-header">
              <h4>Your Promotion Code</h4>
              <p class="code-subtitle">Show this code or enter it at checkout</p>
            </div>
            <div class="code-display-wrapper">
              <div class="promo-code-box">
                <span class="promo-code">{{ voucher.code || voucher.promotion_id }}</span>
              </div>
              <button 
                class="copy-code-btn"
                @click="copyCode"
                :class="{ 'copied': isCopied }"
              >
                <svg v-if="!isCopied" width="18" height="18" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
                  <rect x="9" y="9" width="13" height="13" rx="2" stroke="currentColor" stroke-width="2"/>
                  <path d="M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1" stroke="currentColor" stroke-width="2"/>
                </svg>
                <svg v-else width="18" height="18" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
                  <path d="M20 6L9 17l-5-5" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
                </svg>
                {{ isCopied ? 'Copied!' : 'Copy Code' }}
              </button>
            </div>
          </div>
          
          <div class="voucher-terms">
            <h4>Terms & Conditions</h4>
            <ul>
              <li>Valid for {{ getValidityPeriod() }}</li>
              <li>Cannot be combined with other offers</li>
              <li>Minimum order value may apply</li>
              <li>Valid for dine-in, takeout, and delivery</li>
              <li>One use per customer</li>
            </ul>
          </div>
          
          <div class="modal-actions">
            <button class="remove-voucher-btn" @click="removeVoucher" :disabled="isRemoving">
              <span v-if="!isRemoving" class="btn-content">
                <svg width="20" height="20" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
                  <path d="M3 6h18M8 6V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2m3 0v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6h14zM10 11v6M14 11v6" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
                </svg>
                Remove from Profile
              </span>
              <span v-else class="btn-content">
                <div class="loading-spinner"></div>
                Removing...
              </span>
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'VoucherModal',
  props: {
    voucher: {
      type: Object,
      required: true,
      default: () => ({
        id: 0,
        title: '',
        subtitle: '',
        discount: '',
        code: '',
        qrCode: ''
      })
    },
    isVisible: {
      type: Boolean,
      default: false
    }
  },
  emits: ['close', 'removeVoucher'],
  data() {
    return {
      isCopied: false,
      isRemoving: false
    }
  },
  watch: {
    isVisible(newVal) {
      if (newVal) {
        // Reset state when modal opens
        this.isCopied = false;
        // Prevent body scroll
        document.body.style.overflow = 'hidden';
      } else {
        // Restore body scroll
        document.body.style.overflow = '';
      }
    }
  },
  methods: {
    closeModal() {
      this.$emit('close');
    },
    
    async copyCode() {
      const codeText = this.voucher.code || this.voucher.promotion_id;
      try {
        await navigator.clipboard.writeText(codeText);
        this.isCopied = true;
        setTimeout(() => {
          this.isCopied = false;
        }, 2000);
      } catch (err) {
        // Fallback for older browsers
        const textArea = document.createElement('textarea');
        textArea.value = codeText;
        document.body.appendChild(textArea);
        textArea.select();
        document.execCommand('copy');
        document.body.removeChild(textArea);
        
        this.isCopied = true;
        setTimeout(() => {
          this.isCopied = false;
        }, 2000);
      }
    },
    
    async removeVoucher() {
      this.isRemoving = true;
      
      try {
        // Simulate API call
        await new Promise(resolve => setTimeout(resolve, 1000));
        
        // Remove voucher from localStorage
        const savedVouchers = JSON.parse(localStorage.getItem('ramyeon_saved_vouchers') || '[]');
        const updatedVouchers = savedVouchers.filter(v => v.id !== this.voucher.id);
        localStorage.setItem('ramyeon_saved_vouchers', JSON.stringify(updatedVouchers));
        
        this.$emit('removeVoucher', this.voucher);
        this.closeModal();
        
      } catch (error) {
        console.error('Error removing voucher:', error);
      } finally {
        this.isRemoving = false;
      }
    },
    
    getVoucherDescription() {
      const descriptions = {
        'Welcome Bonus': 'Get 25% off your first order! Perfect way to try our delicious ramyeon.',
        'Shin Ramyun': 'Enjoy 20% off our signature spicy noodle dish - a customer favorite!',
        'Fish Cake': 'Save 15% on our crispy and flavorful fish cake side dish.',
        'Social Signup Bonus': 'Thank you for joining us through social media! Enjoy 30% off.',
        'default': `Enjoy ${this.voucher.discount} on ${this.voucher.title}. Don't miss out on this great deal!`
      };
      
      return descriptions[this.voucher.title] || descriptions.default;
    },
    
    getValidityPeriod() {
      // Generate validity period based on voucher type
      const periods = {
        'Welcome Bonus': '30 days from signup',
        'Shin Ramyun': '14 days from issue',
        'Fish Cake': '7 days from issue',
        'Social Signup Bonus': '30 days from signup',
        'default': '30 days from issue'
      };
      
      return periods[this.voucher.title] || periods.default;
    }
  },
  
  beforeUnmount() {
    // Ensure body scroll is restored if component is destroyed while modal is open
    document.body.style.overflow = '';
  }
}
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.7);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
  padding: 20px;
  backdrop-filter: blur(5px);
  animation: fadeIn 0.3s ease-out;
}

.modal-container {
  background: white;
  border-radius: 20px;
  max-width: 600px;
  width: 100%;
  max-height: 90vh;
  overflow-y: auto;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
  animation: slideUp 0.3s ease-out;
  position: relative;
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 25px 30px 20px;
  border-bottom: 2px solid #f0f0f0;
  background: linear-gradient(135deg, #ff4757, #ff3742);
  color: white;
  border-radius: 20px 20px 0 0;
}

.modal-header h2 {
  margin: 0;
  font-size: 1.8rem;
  font-weight: 700;
}

.close-btn {
  background: rgba(255, 255, 255, 0.2);
  border: none;
  color: white;
  font-size: 1.5rem;
  width: 40px;
  height: 40px;
  border-radius: 50%;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.3s ease;
}

.close-btn:hover {
  background: rgba(255, 255, 255, 0.3);
  transform: scale(1.1);
}

.modal-content {
  padding: 30px;
}

.voucher-details {
  display: flex;
  flex-direction: column;
  gap: 25px;
}

.voucher-info {
  text-align: center;
}

.voucher-info h3 {
  font-size: 1.4rem;
  color: #333;
  margin: 0 0 15px 0;
  font-weight: 600;
}

.discount-badge {
  display: inline-block;
  background: linear-gradient(135deg, #ff4757, #ff3742);
  color: white;
  padding: 12px 25px;
  border-radius: 25px;
  font-size: 1.3rem;
  font-weight: 700;
  margin-bottom: 15px;
  box-shadow: 0 4px 15px rgba(255, 71, 87, 0.3);
}

.voucher-description {
  color: #666;
  font-size: 1rem;
  line-height: 1.6;
  margin: 0;
}

/* Code Section - Modern Design */
.code-section {
  background: linear-gradient(135deg, #fff8f6, #fef5f5);
  border-radius: 16px;
  padding: 30px;
  border: 2px solid rgba(255, 71, 87, 0.1);
  box-shadow: 0 4px 20px rgba(255, 71, 87, 0.08);
}

.code-header {
  text-align: center;
  margin-bottom: 25px;
}

.code-header h4 {
  font-size: 1.4rem;
  color: #333;
  margin: 0 0 8px 0;
  font-weight: 700;
  background: linear-gradient(135deg, #ff4757, #ff3742);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.code-subtitle {
  font-size: 0.9rem;
  color: #666;
  margin: 0;
  font-weight: 500;
}

.code-display-wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
}

.promo-code-box {
  width: 100%;
  background: linear-gradient(135deg, #ff4757, #ff3742);
  padding: 25px 30px;
  border-radius: 16px;
  box-shadow: 0 8px 30px rgba(255, 71, 87, 0.3);
  position: relative;
  overflow: hidden;
}

.promo-code-box::before {
  content: '';
  position: absolute;
  top: -50%;
  right: -50%;
  width: 200%;
  height: 200%;
  background: linear-gradient(45deg, transparent, rgba(255, 255, 255, 0.1), transparent);
  animation: shine 3s infinite;
}

@keyframes shine {
  0% {
    transform: translateX(-100%) translateY(-100%) rotate(45deg);
  }
  100% {
    transform: translateX(100%) translateY(100%) rotate(45deg);
  }
}

.promo-code {
  font-family: 'Courier New', monospace;
  font-size: 2.2rem;
  font-weight: 800;
  color: white;
  letter-spacing: 4px;
  text-transform: uppercase;
  text-align: center;
  display: block;
  text-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
  position: relative;
  z-index: 1;
  word-break: break-all;
}

.copy-code-btn {
  background: white;
  color: #ff4757;
  border: 2px solid #ff4757;
  padding: 14px 28px;
  border-radius: 12px;
  font-weight: 700;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  white-space: nowrap;
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 1rem;
  box-shadow: 0 4px 15px rgba(255, 71, 87, 0.2);
}

.copy-code-btn:hover {
  background: #ff4757;
  color: white;
  transform: translateY(-3px);
  box-shadow: 0 8px 25px rgba(255, 71, 87, 0.4);
}

.copy-code-btn:active {
  transform: translateY(-1px);
}

.copy-code-btn.copied {
  background: #00b894;
  border-color: #00b894;
  color: white;
  transform: scale(1.05);
  box-shadow: 0 6px 20px rgba(0, 184, 148, 0.4);
}

.copy-code-btn svg {
  flex-shrink: 0;
}

.voucher-terms {
  background: #f8f9fa;
  padding: 20px;
  border-radius: 12px;
  border-left: 4px solid #ff4757;
}

.voucher-terms h4 {
  margin: 0 0 15px 0;
  color: #333;
  font-size: 1.1rem;
  font-weight: 600;
}

.voucher-terms ul {
  margin: 0;
  padding-left: 20px;
  color: #666;
  font-size: 0.9rem;
  line-height: 1.6;
}

.voucher-terms li {
  margin-bottom: 5px;
}

.modal-actions {
  display: flex;
  justify-content: center;
  margin-top: 10px;
}

.remove-voucher-btn {
  padding: 16px 40px;
  border: none;
  border-radius: 14px;
  font-weight: 700;
  font-size: 1rem;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  min-width: 200px;
  position: relative;
  overflow: hidden;
  background: linear-gradient(135deg, #ff4757, #ff3742);
  color: white;
  box-shadow: 0 6px 20px rgba(255, 71, 87, 0.3);
}

.btn-content {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
}

.remove-voucher-btn:hover:not(:disabled) {
  background: linear-gradient(135deg, #ff3742, #ff2f3a);
  transform: translateY(-3px) scale(1.02);
  box-shadow: 0 10px 30px rgba(255, 71, 87, 0.5);
}

.remove-voucher-btn:active:not(:disabled) {
  transform: translateY(-1px) scale(1.01);
}

.remove-voucher-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
  transform: none;
  box-shadow: 0 6px 20px rgba(255, 71, 87, 0.2);
}

.loading-spinner {
  width: 16px;
  height: 16px;
  border: 2px solid rgba(255, 255, 255, 0.3);
  border-radius: 50%;
  border-top-color: currentColor;
  animation: spin 1s ease-in-out infinite;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

/* Dark mode support */
.dark-mode .modal-container {
  background: #2d2d2d;
  color: #f5f5f5;
}

.dark-mode .modal-header {
  border-bottom-color: #4a4a4a;
}

.dark-mode .voucher-info h3 {
  color: #f5f5f5;
}

.dark-mode .voucher-description {
  color: #b8b8b8;
}

.dark-mode .code-section {
  background: linear-gradient(135deg, #3a3a3a, #333);
  border-color: rgba(255, 71, 87, 0.2);
}

.dark-mode .code-header h4 {
  color: #f5f5f5;
}

.dark-mode .code-subtitle {
  color: #b8b8b8;
}

.dark-mode .promo-code-box {
  background: linear-gradient(135deg, #ff4757, #ff3742);
}

.dark-mode .copy-code-btn {
  background: #4a4a4a;
  color: #ff4757;
  border-color: #ff4757;
}

.dark-mode .copy-code-btn:hover {
  background: #ff4757;
  color: white;
}

.dark-mode .voucher-terms {
  background: #3a3a3a;
}

.dark-mode .voucher-terms h4 {
  color: #f5f5f5;
}

.dark-mode .voucher-terms ul {
  color: #b8b8b8;
}

.dark-mode .remove-voucher-btn {
  background: linear-gradient(135deg, #ff4757, #ff3742);
}

.dark-mode .remove-voucher-btn:hover:not(:disabled) {
  background: linear-gradient(135deg, #ff3742, #ff2f3a);
}

/* Responsive design */
@media (max-width: 768px) {
  .modal-overlay {
    padding: 10px;
  }
  
  .modal-container {
    max-height: 95vh;
  }
  
  .modal-header {
    padding: 20px 25px 15px;
  }
  
  .modal-header h2 {
    font-size: 1.5rem;
  }
  
  .modal-content {
    padding: 25px 20px;
  }
  
  .voucher-details {
    gap: 20px;
  }
  
  .code-section {
    padding: 20px;
  }
  
  .code-header h4 {
    font-size: 1.2rem;
  }
  
  .promo-code {
    font-size: 1.6rem;
    letter-spacing: 2px;
  }
  
  .promo-code-box {
    padding: 20px;
  }
  
  .copy-code-btn {
    padding: 12px 24px;
    font-size: 0.9rem;
  }
  
  .remove-voucher-btn {
    min-width: auto;
    width: 100%;
    padding: 14px 32px;
  }
}

/* Animations */
@keyframes fadeIn {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

@keyframes slideUp {
  from {
    opacity: 0;
    transform: translateY(50px) scale(0.95);
  }
  to {
    opacity: 1;
    transform: translateY(0) scale(1);
  }
}

/* Scrollbar styling for modal content */
.modal-container::-webkit-scrollbar {
  width: 6px;
}

.modal-container::-webkit-scrollbar-track {
  background: #f1f1f1;
  border-radius: 3px;
}

.modal-container::-webkit-scrollbar-thumb {
  background: #ff4757;
  border-radius: 3px;
}

.modal-container::-webkit-scrollbar-thumb:hover {
  background: #ff3742;
}
</style>
