<template>
  <div class="game-container">
    <header class="game-header">
      <div class="game-info">
        <div class="balance">金币: {{ balance }}</div>
        <div class="streak" v-if="consecutiveWins > 0">连胜: {{ consecutiveWins }}</div>
      </div>
      <div class="events">
        <div class="special-event" v-if="specialEvent">
          {{ specialEvent.message }}
        </div>
        <div class="combinations" v-if="Object.values(combinations).some(c => c.active)">
          <span v-if="combinations.all.active">全盘押注 (x10)</span>
          <span v-if="combinations.diagonal.active">对角押注 (x6)</span>
          <span v-if="combinations.adjacent.active">相邻押注 (x3)</span>
        </div>
        <div class="result" v-if="showResult">
          <span :class="{ 'win': lastWinAmount > 0 }">
            {{ lastWinAmount > 0 ? `赢得 ${lastWinAmount} 金币!` : '很遗憾，这次没有中奖' }}
          </span>
        </div>
      </div>
    </header>
    
    <div class="game-board">
      <div class="betting-area">
        <div v-for="(symbol, index) in symbols" :key="index" 
             class="betting-cell" 
             :class="{ 
               'winner': winnerIndex === index && showResult,
               'spinning': isRolling,
               'glowing': winnerIndex === index && showResult
             }"
             @click="placeBet(index)">
          <div class="bet-amount" v-if="bets[index] > 0">{{ bets[index] }}</div>
          <img :src="`/src/assets/symbols/${symbol}.svg`" :alt="symbol">
        </div>
      </div>
    </div>

    <div class="control-panel">
      <div class="chips">
        <button v-for="chip in chips" 
                :key="chip" 
                class="chip-btn"
                :class="{ active: selectedChip === chip }"
                @click="selectChip(chip)">
          {{ chip }}
        </button>
      </div>
      <button class="action-btn roll" @click="roll" :disabled="!canRoll">开始</button>
      <button class="action-btn clear" @click="clearBets">清空</button>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      balance: 1000,
      selectedChip: 10,
      chips: [10, 100, 1000, 10000, 50000],
      symbols: ['crab', 'fish', 'rooster', 'lobster', 'cat', 'dog'],
      bets: Array(6).fill(0),
      canRoll: false,
      showResult: false,
      winnerIndex: -1,
      lastWinAmount: 0,
      isRolling: false,
      consecutiveWins: 0,
      difficulty: 1,
      specialEvent: null,
      combinations: {
        'all': { active: false, multiplier: 10 },
        'diagonal': { active: false, multiplier: 6 },
        'adjacent': { active: false, multiplier: 3 }
      }
    }
  },
  methods: {
    selectChip(value) {
      this.selectedChip = value
    },
    placeBet(index) {
      // 检查是否正在下注中
      if (this.isRolling) {
        return;
      }
      
      if (this.balance >= this.selectedChip) {
        this.bets[index] += this.selectedChip
        this.balance -= this.selectedChip
        this.canRoll = true
        
        // 检查特殊组合
        this.checkCombinations()
      }
    },
    clearBets() {
      this.balance += this.bets.reduce((a, b) => a + b, 0)
      this.bets = Array(6).fill(0)
      this.canRoll = false
    },
    checkCombinations() {
      // 检查是否押注了所有位置
      this.combinations.all.active = this.bets.every(bet => bet > 0)
      
      // 检查对角线押注
      this.combinations.diagonal.active = (this.bets[0] > 0 && this.bets[5] > 0) || (this.bets[2] > 0 && this.bets[3] > 0)
      
      // 检查相邻押注
      this.combinations.adjacent.active = this.bets.some((bet, i) => 
        bet > 0 && this.bets[(i + 1) % this.bets.length] > 0
      )
    },
    
    triggerSpecialEvent() {
      const events = [
        { type: 'double', chance: 0.1, message: '双倍赔率！' },
        { type: 'half', chance: 0.15, message: '赔率减半...' },
        { type: 'bonus', chance: 0.05, message: '额外奖励！' }
      ]
      
      const random = Math.random()
      let accumulatedChance = 0
      
      for (const event of events) {
        accumulatedChance += event.chance
        if (random < accumulatedChance) {
          this.specialEvent = event
          return
        }
      }
      
      this.specialEvent = null
    },
    
    calculateWinAmount(winner) {
      let baseMultiplier = 3 * this.difficulty
      
      // 特殊组合奖励
      if (this.combinations.all.active) baseMultiplier = this.combinations.all.multiplier
      else if (this.combinations.diagonal.active) baseMultiplier = this.combinations.diagonal.multiplier
      else if (this.combinations.adjacent.active) baseMultiplier = this.combinations.adjacent.multiplier
      
      // 连胜奖励
      const streakBonus = Math.min(this.consecutiveWins * 0.5, 3)
      baseMultiplier += streakBonus
      
      // 特殊事件修正
      if (this.specialEvent) {
        switch(this.specialEvent.type) {
          case 'double': baseMultiplier *= 2; break
          case 'half': baseMultiplier /= 2; break
          case 'bonus': this.balance += 100; break
        }
      }
      
      return Math.floor(this.bets[winner] * baseMultiplier)
    },
    
    async roll() {
      if (this.isRolling) return
      this.isRolling = true
      this.showResult = false
      
      // 触发特殊事件
      this.triggerSpecialEvent()
      
      // 动画效果
      for (let i = 0; i < 20; i++) {
        this.winnerIndex = Math.floor(Math.random() * this.symbols.length)
        await new Promise(resolve => setTimeout(resolve, 100))
      }
      
      // 根据难度调整中奖概率
      const random = Math.random() * this.difficulty
      const winner = Math.floor(random * this.symbols.length)
      this.winnerIndex = winner
      
      // 计算赢得金额
      const winAmount = this.calculateWinAmount(winner)
      this.lastWinAmount = winAmount
      this.balance += winAmount
      
      // 更新连胜次数和难度
      if (winAmount > 0) {
        this.consecutiveWins++
        this.difficulty = Math.min(this.difficulty * 1.1, 2)
      } else {
        this.consecutiveWins = 0
        this.difficulty = Math.max(this.difficulty * 0.9, 1)
      }
      
      // 显示结果
      this.showResult = true
      setTimeout(() => {
        this.bets = Array(6).fill(0)
        this.canRoll = false
        this.isRolling = false
        this.winnerIndex = -1
        this.showResult = false
      }, 3000)
    }
  }
}
</script>

<style>
.game-container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  font-family: Arial, sans-serif;
}

.game-header {
  margin-bottom: 20px;
}

.game-info {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
}

.balance, .streak {
  font-size: 24px;
  font-weight: bold;
}

.events {
  text-align: center;
}

.special-event {
  color: #FFD700;
  font-weight: bold;
  margin-bottom: 5px;
}

.combinations {
  display: flex;
  gap: 10px;
  justify-content: center;
  margin-bottom: 5px;
}

.combinations span {
  background: #4CAF50;
  color: white;
  padding: 2px 8px;
  border-radius: 10px;
  font-size: 14px;
}

.game-board {
  background: #8B0000;
  border-radius: 10px;
  padding: 20px;
  margin-bottom: 20px;
}

.betting-area {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 20px;
  padding: 20px;
}

.betting-cell {
  position: relative;
  background: white;
  border-radius: 10px;
  padding: 20px;
  cursor: pointer;
  text-align: center;
  transition: all 0.3s ease;
  transform-origin: center;
}

@keyframes spin {
  0% { transform: perspective(1000px) rotateY(0deg) scale(1); }
  25% { transform: perspective(1000px) rotateY(90deg) scale(1.1); }
  50% { transform: perspective(1000px) rotateY(180deg) scale(1.15); }
  75% { transform: perspective(1000px) rotateY(270deg) scale(1.1); }
  100% { transform: perspective(1000px) rotateY(360deg) scale(1); }
}

@keyframes glow {
  0% { box-shadow: 0 0 15px rgba(255,215,0,0.6), inset 0 0 5px rgba(255,215,0,0.4); }
  50% { box-shadow: 0 0 30px rgba(255,215,0,0.9), inset 0 0 15px rgba(255,215,0,0.6); }
  100% { box-shadow: 0 0 15px rgba(255,215,0,0.6), inset 0 0 5px rgba(255,215,0,0.4); }
}

@keyframes pulse {
  0% { transform: scale(1); opacity: 1; }
  50% { transform: scale(1.1); opacity: 0.8; }
  100% { transform: scale(1); opacity: 1; }
}

.betting-cell.spinning {
  animation: spin 3s cubic-bezier(0.4, 0, 0.2, 1) infinite;
  transform-style: preserve-3d;
  backface-visibility: hidden;
}

.betting-cell.glowing {
  animation: glow 1.5s ease-in-out infinite;
  background: linear-gradient(45deg, #FFFACD, #FFF8DC);
}

.betting-cell.winner {
  transform: scale(1.15);
  box-shadow: 0 0 30px #FFD700, inset 0 0 20px rgba(255,215,0,0.5);
  background: linear-gradient(135deg, #FFFACD, #FFD700);
  animation: pulse 1.5s ease-in-out infinite;
}

.betting-cell img {
  width: 100%;
  height: auto;
}

.bet-amount {
  position: absolute;
  top: 10px;
  right: 10px;
  background: #FFD700;
  padding: 5px 10px;
  border-radius: 15px;
  font-weight: bold;
}

.control-panel {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 20px;
}

.chips {
  display: flex;
  gap: 10px;
}

.chip-btn {
  padding: 10px 20px;
  border-radius: 20px;
  border: none;
  background: #FFD700;
  cursor: pointer;
  font-weight: bold;
}

.chip-btn.active {
  background: #FFA500;
  color: white;
}

.action-btn {
  padding: 15px 30px;
  border-radius: 25px;
  border: none;
  color: white;
  font-weight: bold;
  cursor: pointer;
}

.action-btn.roll {
  background: #32CD32;
}

.action-btn.clear {
  background: #DC143C;
}

.action-btn:disabled {
  background: #ccc;
  cursor: not-allowed;
}

.result {
  margin-top: 10px;
  font-size: 20px;
  font-weight: bold;
  text-align: center;
}

.result .win {
  color: #FFD700;
  text-shadow: 1px 1px 2px rgba(0,0,0,0.5);
}
</style>