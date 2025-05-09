<script setup>
import { useMessage } from 'naive-ui'
import { ref } from 'vue'
import LogPanel from '../components/LogPanel.vue'
import { CombatEntity, CombatManager, CombatType, generateEnemy } from '../plugins/combat'
import { getRandomOptions } from '../plugins/dungeon'
import dungeonBuffs from '../plugins/dungeonBuffs'
import { getRealmName } from '../plugins/realm'
import { usePlayerStore } from '../stores/player'

const playerStore = usePlayerStore()
const message = useMessage()
const logRef = ref(null)
const playerAttacking = ref(false)
const playerHurt = ref(false)
const enemyAttacking = ref(false)
const enemyHurt = ref(false)
const infoShow = ref(false)
const infoType = ref('')

const floorData = computed(() => {
  switch (playerStore.dungeonDifficulty) {
    case 1:
      return playerStore.dungeonHighestFloor
    case 2:
      return playerStore.dungeonHighestFloor_2
    case 5:
      return playerStore.dungeonHighestFloor_5
    case 10:
      return playerStore.dungeonHighestFloor_10
    case 100:
      return playerStore.dungeonHighestFloor_100
    default:
      return playerStore.dungeonHighestFloor
  }
})

// 副本状态
const dungeonState = ref({
  floor: floorData.value,
  inCombat: false,
  showingOptions: false,
  currentOptions: [],
  combatManager: null
})


// 当前战斗日志
const combatLog = ref([])

// 根据选项类型获取颜色
const getOptionColor = (type) => {
  const types = {
    epic: {
      name: '史诗',
      color: '#e91e63'
    },
    rare: {
      name: '稀有',
      color: '#2196f3'
    },
    common: {
      name: '普通',
      color: '#4caf50'
    }
  }
  return types[type]
}

// 创建玩家战斗实体
const createPlayerEntity = () => {
  // 基础属性
  const baseStats = {
    health: playerStore.baseAttributes.health,
    damage: playerStore.baseAttributes.attack,
    defense: playerStore.baseAttributes.defense,
    speed: playerStore.baseAttributes.speed,
    // 战斗属性
    critRate: playerStore.combatAttributes.critRate,
    comboRate: playerStore.combatAttributes.comboRate,
    counterRate: playerStore.combatAttributes.counterRate,
    stunRate: playerStore.combatAttributes.stunRate,
    dodgeRate: playerStore.combatAttributes.dodgeRate,
    vampireRate: playerStore.combatAttributes.vampireRate,
    // 战斗抗性
    critResist: playerStore.combatResistance.critResist,
    comboResist: playerStore.combatResistance.comboResist,
    counterResist: playerStore.combatResistance.counterResist,
    stunResist: playerStore.combatResistance.stunResist,
    dodgeResist: playerStore.combatResistance.dodgeResist,
    vampireResist: playerStore.combatResistance.vampireResist,
    // 特殊属性
    healBoost: playerStore.specialAttributes.healBoost,
    critDamageBoost: playerStore.specialAttributes.critDamageBoost,
    critDamageReduce: playerStore.specialAttributes.critDamageReduce,
    finalDamageBoost: playerStore.specialAttributes.finalDamageBoost,
    finalDamageReduce: playerStore.specialAttributes.finalDamageReduce,
    combatBoost: playerStore.specialAttributes.combatBoost,
    resistanceBoost: playerStore.specialAttributes.resistanceBoost,
    // 其他属性
    spiritDamage: playerStore.spirit * 0.1,
    maxHealth: playerStore.baseAttributes.health
  }
  const entity = new CombatEntity(playerStore.name, playerStore.level, baseStats, playerStore.realm)
  // 应用所有激活的增益效果
  const activeBuffs = dungeonBuffs.getActiveBuffs()
  activeBuffs.forEach(buff => {
    if (typeof buff.effect === 'function') {
      buff.effect(entity)
    }
  })
  return entity
}

// 开始新的副本
const startDungeon = () => {
  const startingFloor = floorData.value
  dungeonState.value = {
    floor: startingFloor,
    inCombat: false,
    showingOptions: false,
    currentOptions: [],
    combatManager: null
  }
  playerStore.dungeonTotalRuns++  // 增加总探索次数
  nextFloor()
}

// 进入下一层
const nextFloor = () => {
  dungeonState.value = {
    ...dungeonState.value,
    floor: dungeonState.value.floor + 1
  }
  const floor = dungeonState.value.floor
  // 检查是否需要显示选项
  if (floor === 1 || floor % 5 === 0) {
    const randRefres = Math.floor(Math.random() * 3) + 1 
    message.success(`获得了${randRefres}刷新次数`)
    refreshNumber.value = randRefres
    showOptions()
    return
  }
  startCombat()
}

// 显示随机选项
const showOptions = () => {
  dungeonState.value.showingOptions = true
  dungeonState.value.currentOptions = getRandomOptions(dungeonState.value.floor)
}

// 选择选项
const selectOption = (option) => {
  dungeonBuffs.apply(playerStore, option)
  message.success(`选择了：${option.name}`)
  dungeonState.value.showingOptions = false
  dungeonState.value.currentOptions = []
  startCombat()
}

// 处理失败
const handleDefeat = () => {
  dungeonState.value.inCombat = false
  infoShow.value = false
  infoType.value = ''
  message.error(`在第 ${dungeonState.value.floor} 层被击败了...`)
  playerStore.dungeonDeathCount++
  // 清除所有临时增益效果
  dungeonBuffs.clear(playerStore)
  // 记录失败层数
  playerStore.dungeonLastFailedFloor = dungeonState.value.floor
  // 随机跌落境界或修为
  if (playerStore.dungeonDifficulty !== 100) {
    // 损失一定修为值作为惩罚
    const cultivationLossRate = Math.random() * 0.4 + 0.1 // 随机10%到50%
    const cultivationLoss = Math.floor(playerStore.cultivation * cultivationLossRate)
    playerStore.cultivation = Math.max(0, playerStore.cultivation - cultivationLoss)
    message.error(`战斗失败！损失了${cultivationLoss}点修为。`)
  } else {
    // 跌落境界作为惩罚
    const randomGradeLoss = Math.floor(Math.random() * 3) + 1 // 随机损失1-3个境界
    const playerLevel = Math.max(1, playerStore.level - randomGradeLoss) // 降低境界
    playerStore.level = playerLevel
    playerStore.cultivation = 0; // 移除所有灵力
    playerStore.maxCultivation = getRealmName(playerLevel).maxCultivation // 降低所需最大灵力值
    message.error(`战斗失败！跌落了${playerLevel}个境界。`)
  }
}

// 开始战斗
const startCombat = () => {
  const floor = dungeonState.value.floor
  const isBossFloor = floor % 10 === 0
  const isEliteFloor = floor % 5 === 0
  const enemyType = isBossFloor ? CombatType.BOSS : isEliteFloor ? CombatType.ELITE : CombatType.NORMAL
  // 创建玩家战斗实体，并应用所有增益效果
  const playerEntity = createPlayerEntity()
  // 创建敌人
  const enemy = generateEnemy(floor, enemyType, playerStore.dungeonDifficulty)
  // 创建战斗管理器
  dungeonState.value.combatManager = new CombatManager(
    playerEntity,
    enemy,
    (log) => {
      if (logRef.value) {
        logRef.value.addLog(log)
      }
    }
  )
  dungeonState.value.inCombat = true
  dungeonState.value.combatManager.start() // 初始化战斗状态
  autoCombat() // 开始自动战斗
}

const sleep = (ms) => new Promise(resolve => setTimeout(resolve, ms));

const autoCombat = async () => {
  while (dungeonState.value.inCombat) {
    const { combatManager } = dungeonState.value;
    const turnResult = combatManager.executeTurn();
    const combatLog = combatManager.getCombatLog();

    switch (turnResult.state) {
      case 'victory':
        handleVictory();
        break;
      case 'defeat':
        handleDefeat();
        break;
    }

    for (const turn of turnResult.results) {
      if (!turn) {break;}

      const isPlayerTurn = turn.attacker === combatManager.player?.name;

      if (isPlayerTurn) {
        playerAttacking.value = true;
        enemyHurt.value = true;
      } else {
        enemyAttacking.value = true;
        playerHurt.value = true;
      }

      await sleep(500);

      // Reset animation
      playerAttacking.value = false;
      enemyHurt.value = false;
      enemyAttacking.value = false;
      playerHurt.value = false;

      await sleep(500);
    }

    // Combat log
    combatLog.forEach((log) => {
      logRef.value?.addLog('info', log);
    });

    await sleep(500);
  }
};

// 处理胜利
const handleVictory = () => {
  dungeonState.value.inCombat = false
  message.success(`击败了第 ${dungeonState.value.floor} 层的敌人！`)
  // 更新统计数据
  playerStore.dungeonTotalKills++
  if (dungeonState.value.floor % 10 === 0) {
    playerStore.dungeonBossKills++
  } else if (dungeonState.value.floor % 5 === 0) {
    // 增加洗练石
    playerStore.refinementStones += playerStore.dungeonDifficulty
    playerStore.dungeonEliteKills++
    message.success(`获得了${playerStore.dungeonDifficulty}颗洗练石`)
  }
  // 更新最高层数记录
  if (dungeonState.value.floor > playerStore.dungeonHighestFloor) {
    playerStore.dungeonHighestFloor = dungeonState.value.floor
  }
  // 获得奖励
  const rewards = generateRewards()
  rewards.forEach(reward => {
    playerStore.spiritStones += reward.amount
    message.success(`获得了 ${reward.amount} 灵石！`)
    playerStore.dungeonTotalRewards++
  })
  // 进入下一层
  nextFloor()
}

// 生成奖励
const generateRewards = () => {
  const rewards = []
  // 灵石奖励
  const baseStones = 10 * dungeonState.value.floor * playerStore.dungeonDifficulty
  rewards.push({
    type: 'spirit_stones',
    amount: baseStones
  })
  return rewards
}

const infoCliclk = (type) => {
  infoShow.value = true
  infoType.value = type
}

const dungeonOptions = [
  {
    label: '简单',
    value: 1
  },
  {
    label: '普通',
    value: 2
  },
  {
    label: '困难',
    value: 5
  },
  {
    label: '地狱',
    value: 10
  },
  {
    label: '通天',
    value: 100
  }
]

const handleUpdateValue = (value, option) => {
  if (value === 100) {
    message.warning('警告! 通天难度挑战失败后会跌落境界')
  }
}
const refreshNumber = ref(3)
// 刷新选择
const handleRefreshOptions = () => {
  showOptions()
  refreshNumber.value--
}

</script>

<template>
  <div class="dungeon-container">
    <n-card title="秘境探索">
      <template #header-extra>
        <n-space>
          <n-select v-model:value="playerStore.dungeonDifficulty" @update:value="handleUpdateValue" placeholder="请选择难度"
            :options="dungeonOptions" style="width: 120px"
            :disabled="dungeonState.inCombat || dungeonState.showingOptions" />
          <n-button type="primary" @click="startDungeon"
            :disabled="dungeonState.inCombat || dungeonState.showingOptions">
            开始探索
          </n-button>
        </n-space>
      </template>
      <n-space vertical>
        <!-- 层数显示 -->
        <n-statistic label="当前层数" :value="dungeonState.floor" />
        <!-- 选项界面 -->
        <n-card v-if="dungeonState.showingOptions" title="选择增益">
          <template #header-extra>
            <n-space>
              <n-button type="primary" @click="handleRefreshOptions" :disabled="refreshNumber === 0">
                刷新增益({{refreshNumber}})
              </n-button>
            </n-space>
          </template>
          <div class="option-cards">
            <div v-for="option in dungeonState.currentOptions" :key="option.id" class="option-card"
              :style="{ borderColor: getOptionColor(option.type).color }" @click="selectOption(option)">
              <div class="option-name">{{ option.name }}</div>
              <div class="option-description">{{ option.description }}</div>
              <div class="option-quality" :style="{ color: getOptionColor(option.type).color }">
                {{ getOptionColor(option.type).name }}
              </div>
            </div>
          </div>
        </n-card>
        <!-- 战斗界面 -->
        <template v-if="dungeonState.inCombat && dungeonState.combatManager">
          <n-card :bordered="false">
            <n-divider>{{ dungeonState.combatManager.round }} / {{ dungeonState.combatManager.maxRounds }}回合</n-divider>
            <!-- 添加战斗场景 -->
            <div class="combat-scene">
              <div class="character player" :class="{ 'attack': playerAttacking, 'hurt': playerHurt }">
                <div v-if="playerAttacking" class="attack-effect player-effect"></div>
                <n-button class="character-name" type="info" dashed @click="infoCliclk('player')">
                  {{ dungeonState.combatManager.player.name }}
                </n-button>
                <div class="character-avatar player-avatar">
                  {{ dungeonState.combatManager.player.name[0] }}
                </div>
                <div class="health-bar">
                  <div class="health-fill"
                    :style="{ width: `${(dungeonState.combatManager.player.currentHealth / dungeonState.combatManager.player.stats.maxHealth) * 100}%` }">
                  </div>
                </div>
              </div>
              <div class="character enemy" :class="{ 'attack': enemyAttacking, 'hurt': enemyHurt }">
                <div v-if="enemyAttacking" class="attack-effect enemy-effect"></div>
                <n-button class="character-name" type="error" dashed @click="infoCliclk('enemy')">
                  {{ dungeonState.combatManager.enemy.name }}
                </n-button>
                <div class="character-avatar enemy-avatar">
                  {{ dungeonState.combatManager.enemy.name[0] }}
                </div>
                <div class="health-bar">
                  <div class="health-fill"
                    :style="{ width: `${(dungeonState.combatManager.enemy.currentHealth / dungeonState.combatManager.enemy.stats.maxHealth) * 100}%` }">
                  </div>
                </div>
              </div>
            </div>
            <n-modal v-model:show="infoShow" preset="dialog"
              :title="`${infoType == 'player' ? dungeonState.combatManager.player.name : dungeonState.combatManager.enemy.name }的属性`">
              <n-card :bordered="false">
                <!-- 玩家属性 -->
                <template v-if="infoType == 'player'">
                  <n-divider>基础属性</n-divider>
                  <n-descriptions bordered :column="2">
                    <n-descriptions-item label="生命值">
                      {{ dungeonState.combatManager.player.currentHealth.toFixed(1) }} /
                      {{ dungeonState.combatManager.player.stats.maxHealth.toFixed(1) }}
                    </n-descriptions-item>
                    <n-descriptions-item label="攻击力">
                      {{ dungeonState.combatManager.player.stats.damage.toFixed(1) }}
                    </n-descriptions-item>
                    <n-descriptions-item label="防御力">
                      {{ dungeonState.combatManager.player.stats.defense.toFixed(1) }}
                    </n-descriptions-item>
                    <n-descriptions-item label="速度">
                      {{ dungeonState.combatManager.player.stats.speed.toFixed(1) }}
                    </n-descriptions-item>
                  </n-descriptions>
                  <n-divider>战斗属性</n-divider>
                  <n-descriptions bordered :column="3">
                    <n-descriptions-item label="暴击率">
                      {{ (dungeonState.combatManager.player.stats.critRate * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="连击率">
                      {{ (dungeonState.combatManager.player.stats.comboRate * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="反击率">
                      {{ (dungeonState.combatManager.player.stats.counterRate * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="眩晕率">
                      {{ (dungeonState.combatManager.player.stats.stunRate * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="闪避率">
                      {{ (dungeonState.combatManager.player.stats.dodgeRate * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="吸血率">
                      {{ (dungeonState.combatManager.player.stats.vampireRate * 100).toFixed(1) }}%
                    </n-descriptions-item>
                  </n-descriptions>
                  <n-divider>战斗抗性</n-divider>
                  <n-descriptions bordered :column="3">
                    <n-descriptions-item label="抗暴击">
                      {{ (dungeonState.combatManager.player.stats.critResist * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="抗连击">
                      {{ (dungeonState.combatManager.player.stats.comboResist * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="抗反击">
                      {{ (dungeonState.combatManager.player.stats.counterResist * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="抗眩晕">
                      {{ (dungeonState.combatManager.player.stats.stunResist * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="抗闪避">
                      {{ (dungeonState.combatManager.player.stats.dodgeResist * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="抗吸血">
                      {{ (dungeonState.combatManager.player.stats.vampireResist * 100).toFixed(1) }}%
                    </n-descriptions-item>
                  </n-descriptions>
                  <n-divider>特殊属性</n-divider>
                  <n-descriptions bordered :column="4">
                    <n-descriptions-item label="强化治疗">
                      {{ (dungeonState.combatManager.player.stats.healBoost * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="强化爆伤">
                      {{ (dungeonState.combatManager.player.stats.critDamageBoost * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="弱化爆伤">
                      {{ (dungeonState.combatManager.player.stats.critDamageReduce * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="最终增伤">
                      {{ (dungeonState.combatManager.player.stats.finalDamageBoost * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="最终减伤">
                      {{ (dungeonState.combatManager.player.stats.finalDamageReduce * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="战斗属性提升">
                      {{ (dungeonState.combatManager.player.stats.combatBoost * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="战斗抗性提升">
                      {{ (dungeonState.combatManager.player.stats.resistanceBoost * 100).toFixed(1) }}%
                    </n-descriptions-item>
                  </n-descriptions>
                </template>
                <!-- 敌人属性 -->
                <template v-else>
                  <n-divider>基础属性</n-divider>
                  <n-descriptions bordered :column="2">
                    <n-descriptions-item label="生命值">
                      {{ dungeonState.combatManager.enemy.currentHealth.toFixed(1) }} /
                      {{ dungeonState.combatManager.enemy.stats.maxHealth.toFixed(1) }}
                    </n-descriptions-item>
                    <n-descriptions-item label="攻击力">
                      {{ dungeonState.combatManager.enemy.stats.damage.toFixed(1) }}
                    </n-descriptions-item>
                    <n-descriptions-item label="防御力">
                      {{ dungeonState.combatManager.enemy.stats.defense.toFixed(1) }}
                    </n-descriptions-item>
                    <n-descriptions-item label="速度">
                      {{ dungeonState.combatManager.enemy.stats.speed.toFixed(1) }}
                    </n-descriptions-item>
                  </n-descriptions>
                  <n-divider>战斗属性</n-divider>
                  <n-descriptions bordered :column="3">
                    <n-descriptions-item label="暴击率">
                      {{ (dungeonState.combatManager.enemy.stats.critRate * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="连击率">
                      {{ (dungeonState.combatManager.enemy.stats.comboRate * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="反击率">
                      {{ (dungeonState.combatManager.enemy.stats.counterRate * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="眩晕率">
                      {{ (dungeonState.combatManager.enemy.stats.stunRate * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="闪避率">
                      {{ (dungeonState.combatManager.enemy.stats.dodgeRate * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="吸血率">
                      {{ (dungeonState.combatManager.enemy.stats.vampireRate * 100).toFixed(1) }}%
                    </n-descriptions-item>
                  </n-descriptions>
                  <n-divider>战斗抗性</n-divider>
                  <n-descriptions bordered :column="3">
                    <n-descriptions-item label="抗暴击">
                      {{ (dungeonState.combatManager.enemy.stats.critResist * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="抗连击">
                      {{ (dungeonState.combatManager.enemy.stats.comboResist * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="抗反击">
                      {{ (dungeonState.combatManager.enemy.stats.counterResist * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="抗眩晕">
                      {{ (dungeonState.combatManager.enemy.stats.stunResist * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="抗闪避">
                      {{ (dungeonState.combatManager.enemy.stats.dodgeResist * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="抗吸血">
                      {{ (dungeonState.combatManager.enemy.stats.vampireResist * 100).toFixed(1) }}%
                    </n-descriptions-item>
                  </n-descriptions>
                  <n-divider>特殊属性</n-divider>
                  <n-descriptions bordered :column="3">
                    <n-descriptions-item label="强化治疗">
                      {{ (dungeonState.combatManager.enemy.stats.healBoost * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="强化爆伤">
                      {{ (dungeonState.combatManager.enemy.stats.critDamageBoost * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="弱化爆伤">
                      {{ (dungeonState.combatManager.enemy.stats.critDamageReduce * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="最终增伤">
                      {{ (dungeonState.combatManager.enemy.stats.finalDamageBoost * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="最终减伤">
                      {{ (dungeonState.combatManager.enemy.stats.finalDamageReduce * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="战斗属性提升">
                      {{ (dungeonState.combatManager.enemy.stats.combatBoost * 100).toFixed(1) }}%
                    </n-descriptions-item>
                    <n-descriptions-item label="战斗抗性提升">
                      {{ (dungeonState.combatManager.enemy.stats.resistanceBoost * 100).toFixed(1) }}%
                    </n-descriptions-item>
                  </n-descriptions>
                </template>
              </n-card>
            </n-modal>
            <!-- 战斗日志 -->
            <log-panel ref="logRef" :messages="combatLog" style="margin-top: 16px" />
          </n-card>
        </template>
      </n-space>
    </n-card>
  </div>
</template>

<style scoped>
.dungeon-container {
  margin: 0 auto;
}

.option-cards {
  display: flex;
  gap: 16px;
  padding: 16px;
  margin: 0 auto;
}

.option-card {
  position: relative;
  padding: 20px;
  border: 2px solid;
  border-radius: 12px;
  background: var(--n-color);
  cursor: pointer;
  transition: all 0.3s ease;
  display: flex;
  flex-direction: column;
  min-height: 100px;
  width: 33%;
}

.option-card:hover {
  transform: translateX(5px);
  box-shadow: 4px 4px 12px rgba(0, 0, 0, 0.1);
}

.option-name {
  font-size: 1.3em;
  font-weight: bold;
  margin-bottom: 12px;
  padding-right: 80px;
}

.option-description {
  flex-grow: 1;
  font-size: 1em;
  color: var(--n-text-color);
  line-height: 1.6;
  margin-bottom: 8px;
}

.option-quality {
  position: absolute;
  top: 20px;
  right: 20px;
  font-size: 0.9em;
  font-weight: bold;
  padding: 4px 12px;
  border-radius: 20px;
  background: var(--n-color);
}

.combat-scene {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 20px;
  margin-bottom: 20px;
  min-height: 200px;
  background: rgba(0, 0, 0, 0.05);
  border-radius: 8px;
}

.character {
  display: flex;
  flex-direction: column;
  align-items: center;
  transition: transform 0.3s ease;
}

.character-avatar {
  font-size: 48px;
  margin: 10px 0;
}

.character-name {
  font-weight: bold;
  margin-bottom: 8px;
}

.health-bar {
  width: 100px;
  height: 10px;
  background: #ff000033;
  border-radius: 5px;
  overflow: hidden;
}

.health-fill {
  height: 100%;
  background: #ff0000;
  transition: width 0.3s ease;
}

.character.attack {
  animation: attack 0.5s ease;
}

.character.hurt {
  animation: hurt 0.5s ease;
}

.character-avatar {
  width: 60px;
  height: 60px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 24px;
  font-weight: bold;
  margin: 10px 0;
  color: #fff;
}

.player-avatar {
  background: linear-gradient(135deg, #4caf50, #2196f3);
  border-radius: 12px;
}

.enemy-avatar {
  background: linear-gradient(135deg, #ff5722, #e91e63);
  clip-path: polygon(50% 0%, 100% 38%, 100% 100%, 0 100%, 0% 38%);
}

.attack-effect {
  position: absolute;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  pointer-events: none;
}

.player-effect {
  background: radial-gradient(circle, #4caf50, #2196f3);
  animation: player-attack-effect 0.5s ease-out;
  right: -10px;
}

.enemy-effect {
  background: radial-gradient(circle, #ff5722, #e91e63);
  animation: enemy-attack-effect 0.5s ease-out;
  left: -10px;
}

.enemy.attack {
  animation: enemy-attack 0.5s ease;
}

@keyframes player-attack-effect {
  0% {
    transform: scale(0.5) translateX(0);
    opacity: 1;
  }
  100% {
    transform: scale(1.5) translateX(200px);
    opacity: 0;
  }
}

@keyframes enemy-attack-effect {
  0% {
    transform: scale(0.5) translateX(0);
    opacity: 1;
  }
  100% {
    transform: scale(1.5) translateX(-200px);
    opacity: 0;
  }
}

@keyframes attack {
  0% {
    transform: translateX(0) rotate(0deg);
  }
  25% {
    transform: translateX(20px) rotate(5deg);
  }
  50% {
    transform: translateX(40px) rotate(0deg);
  }
  75% {
    transform: translateX(20px) rotate(-5deg);
  }
  100% {
    transform: translateX(0) rotate(0deg);
  }
}

@keyframes hurt {
  0% {
    transform: translateX(0);
  }
  25% {
    transform: translateX(-10px);
  }
  75% {
    transform: translateX(10px);
  }
  100% {
    transform: translateX(0);
  }
}

@keyframes enemy-attack {
  0% {
    transform: translateX(0) rotate(0deg);
  }
  25% {
    transform: translateX(-20px) rotate(-5deg);
  }
  50% {
    transform: translateX(-40px) rotate(0deg);
  }
  75% {
    transform: translateX(-20px) rotate(5deg);
  }
  100% {
    transform: translateX(0) rotate(0deg);
  }
}
</style>
