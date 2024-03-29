<template>
   <div class="resultsContainer">
      <h2>results</h2>
      <div class="centered chart">
         <div class="graphSettingsContainer">
            <div class="modeButtonsContainer">
               <button class="button modeButton centered" @click="strikeType = 'averageStrike'" :class="{selected: strikeType === 'averageStrike'}">average</button>
               <button class="button modeButton centered" @click="strikeType = 'nonCriticalStrike'" :class="{selected: strikeType === 'nonCriticalStrike'}">non-critical</button>
               <button class="button modeButton centered" @click="strikeType = 'criticalStrike'" :class="{selected: strikeType === 'criticalStrike'}">critical</button>
               <div class="modeButtonHighlight" :class="strikeType"/>
               <span class="strikeTooltip" v-if="showModeTooltips">choose the strike type, meaning graph will display damage of regular/critical/averaged strike</span>
            </div>
            <div class="modeButtonsContainer">
               <button class="button modeButton centered" @click="levelSetting = 'Equal'" :class="{selected: levelSetting === 'Equal'}">equal</button>
               <button class="button modeButton centered" @click="levelSetting = 'SelectedChanging'" :class="{selected: levelSetting === 'SelectedChanging'}">selected/changing</button>
               <button class="button modeButton centered" @click="levelSetting = 'ChangingSelected'" :class="{selected: levelSetting === 'ChangingSelected'}">changing/selected</button>
               <div class="modeButtonHighlight" :class="levelSetting"/>
               <span class="levelTooltip" v-if="showModeTooltips">choose the level setting the strike damage is shown for.<br/>equal means 1-18 vs 1-18<br/>selected vs changing means {{level(true)}} vs 1-18<br/>changing vs selected means 1-18 vs {{level(false)}}</span>
            </div>
         </div>
         <div class="chartContainer">
            <canvas class="damageChart" ref="canvas"></canvas>
         </div>
      </div>
   </div>
</template>

<script>
import Chart from 'chart.js/auto';
Chart.defaults.elements.point.hoverRadius = 5;
const chartGridLineColor = 'hsla(0, 0%, 100%, 0.1)';
const chartGridTextColor = 'hsla(0, 0%, 100%, 0.6)';

export default {
   name: 'results',
   data(){
      return {
         strikeType: "averageStrike",
         levelSetting: "Equal",
         graphColors: [265, 120, 0]
      }
   },
   mounted(){
      const labels = Array.from({length: 18}, (_, index) => index + 1)
      const data = {
         labels: labels,
         datasets: JSON.parse(JSON.stringify(this.datasetsToDisplay))
      }
      this.chart = new Chart(this.$refs.canvas, {type: 'line', data: data, options: {
         aspectRatio: (1 + Math.sqrt(5)) / 2,
         interaction: {
            mode: 'index',
            intersect: false,
         },
         scales: {
            x: {
               grid: {color: chartGridLineColor},
               ticks: {color: chartGridTextColor}
            },
            y: {
               grid: {color: chartGridLineColor},
               ticks: {color: chartGridTextColor}
            }
         },
         plugins: {
            legend: {
               labels: {
                  boxWidth: 15,
                  boxHeight: 15,
                  color: chartGridTextColor,
               },
            },
            tooltip: {
               position: 'nearest',
               backgroundColor: 'hsla(0, 0%, 100%, 0.9)',
               titleColor: 'hsla(0, 0%, 0%, 1)',
               titleFont: {size: 14},
               bodyColor: 'hsla(0, 0%, 0%, 1)',
               bodyFont: {size: 12},
               callbacks: {
                  title: (context) => {
                     return this.levelSetting === "Equal" ? `${context[0].label} vs ${context[0].label}` : this.levelSetting === "SelectedChanging" ? `${this.level(true)} vs ${context[0].label}` : `${context[0].label} vs ${this.level(false)}`
                  }
               },
            }
         },
      }})
   },
   methods: {
      updateChart(){
         if(this.chart.data.datasets.length != this.datasetsToDisplay.length){
            this.chart.data.datasets = JSON.parse(JSON.stringify(this.datasetsToDisplay))
         } else{
            this.datasetsToDisplay.forEach((dataset, datasetIndex) => {
               this.chart.data.datasets[datasetIndex].data = dataset.data
            })
         }
         this.chart.update()
      },
      calculateDamage(mainStats, mainItems, mainLevel, targetLevel){
         let attackDamageModifier = this.champion(true).id === "Kalista" ? 0.9 : 1
         var targetHealth = this.targetStats.health[targetLevel - 1]
         let basicAttackDamageModifier = this.selectedItems(false).includes("3047") ? 0.88 : 1

         // var targetHealth = 1000
         const healthDifference = ((targetHealth - mainStats.health[mainLevel - 1]) / 100) > 20 ? 20 : ((targetHealth - mainStats.health[mainLevel - 1]) / 100) < 0 ? 0 : ((targetHealth - mainStats.health[mainLevel - 1]) / 100)
         const physicalDamageModifier = (mainItems.includes("3036") ? ((healthDifference * 0.0075) + 1) : 1)

         const criticalStrikeDamageModifier = mainItems.includes("3124") ? 1 : mainStats.criticalStrike[1] / 100
         const magicDamageModifier = 1

         const nonCriticalStrike = {
            physicalDamage: ((mainStats.attackDamage[mainLevel - 1] * attackDamageModifier * (this.champion(true).id === "Corki" ? 0.2 : 1) * basicAttackDamageModifier) + this.onHitDamage(mainStats, mainLevel, mainItems)[0]) * physicalDamageModifier,
            magicDamage: (this.onHitDamage(mainStats, mainLevel, mainItems)[1] + (mainStats.attackDamage[mainLevel - 1] * (this.champion(true).id === "Corki" ? 0.8 : 0) * basicAttackDamageModifier)) * magicDamageModifier
         }
         const criticalStrike = {
            physicalDamage: ((mainStats.attackDamage[mainLevel - 1] * attackDamageModifier * (this.champion(true).id === "Corki" ? 0.2 : 1) * criticalStrikeDamageModifier * basicAttackDamageModifier) + this.onHitDamage(mainStats, mainLevel, mainItems)[0]) * physicalDamageModifier,
            magicDamage: (this.onHitDamage(mainStats, mainLevel, mainItems)[1] + (mainStats.attackDamage[mainLevel - 1] * criticalStrikeDamageModifier * (this.champion(true).id === "Corki" ? 0.8 : 0) * basicAttackDamageModifier)) * magicDamageModifier
         }
         const averageStrike = {
            physicalDamage: ((mainStats.attackDamage[mainLevel - 1] * attackDamageModifier * basicAttackDamageModifier * (this.champion(true).id === "Corki" ? 0.2 : 1) * (1 + ((mainStats.criticalStrike[0] / 100) * (criticalStrikeDamageModifier - 1)))) + this.onHitDamage(mainStats, mainLevel, mainItems)[0]) * physicalDamageModifier,
            magicDamage: (this.onHitDamage(mainStats, mainLevel, mainItems)[1] + (mainStats.attackDamage[mainLevel - 1] * (1 + ((mainStats.criticalStrike[0] / 100) * (criticalStrikeDamageModifier - 1))) * basicAttackDamageModifier * (this.champion(true).id === "Corki" ? 0.8 : 0))) * magicDamageModifier
         }
         var effectiveArmor = (this.targetStats.armor[targetLevel - 1] - (mainStats.armorPenetration[0] * (0.6 + (0.4 * mainLevel / 18)))) * (1 - (mainStats.armorPenetration[1] / 100))
         // var effectiveArmor = (0 - (mainStats.armorPenetration[0] * (0.6 + (0.4 * mainLevel / 18)))) * (1 - (mainStats.armorPenetration[1] / 100))
         effectiveArmor = effectiveArmor < 0 ? 0 : effectiveArmor
         var effectiveMagicResists = (this.targetStats.magicResists[targetLevel - 1] - mainStats.magicPenetration[0]) * (1 -(mainStats.magicPenetration[1] / 100))
         // var effectiveMagicResists = (0 - mainStats.magicPenetration[0]) * (1 -(mainStats.magicPenetration[1] / 100))
         effectiveMagicResists = effectiveMagicResists < 0 ? 0 : effectiveMagicResists

         return {
            nonCriticalStrike: {
               physicalDamage: nonCriticalStrike.physicalDamage * (100 / (100 + effectiveArmor)),
               magicDamageModifier: nonCriticalStrike.magicDamage * (100 / (100 + effectiveMagicResists)),
               totalDamage: (nonCriticalStrike.physicalDamage * (100 / (100 + effectiveArmor))) + (nonCriticalStrike.magicDamage * (100 / (100 + effectiveMagicResists)))
            },
            criticalStrike: {
               physicalDamage: criticalStrike.physicalDamage * (100 / (100 + effectiveArmor)),
               magicDamageModifier: criticalStrike.magicDamage * (100 / (100 + effectiveMagicResists)),
               totalDamage: (criticalStrike.physicalDamage * (100 / (100 + effectiveArmor))) + (criticalStrike.magicDamage * (100 / (100 + effectiveMagicResists)))
            },
            averageStrike: {
               physicalDamage: averageStrike.physicalDamage * (100 / (100 + effectiveArmor)),
               magicDamageModifier: averageStrike.magicDamage * (100 / (100 + effectiveMagicResists)),
               totalDamage: (averageStrike.physicalDamage * (100 / (100 + effectiveArmor))) + (averageStrike.magicDamage * (100 / (100 + effectiveMagicResists)))
            }
         }
      },
      selectedItems(isMain){
         return this.$store.getters.getSelectedItems(isMain)
      },
      legendaries(isMain){
         return this.$store.getters.getSelectedItems(isMain).filter(item => (!this.mythics.includes(item) && this.$store.getters.getItem(item).gold.total >= 1600) || item === "3112" || item === "2051" || item === "3184" || item === "3177")
      },
      calculateStat(base, level, growth){
         return base + (growth * (level - 1) * (0.7025 + (0.0175 * (level - 1))))
      },
      champion(isMain){
         return isMain ? this.$store.getters.getMainChampion : this.$store.getters.getTargetChampion
      },
      isRanged(isMain){
         return this.champion(isMain).stats.attackrange >= 300
      },
      level(isMain){
         return this.$store.getters.getLevel(isMain)
      },
      onHitDamage(stats, level, items){ // [physical, magic]
         const onHitGuinsoo = Math.floor(stats.criticalStrike[0] / 5) * (items.includes("3124") ? 40 : items.includes("6677") ? 35 : 0) / 4

         const onHitTitanic = items.includes("3748") ? ((this.isRanged(true) ? 3 : 4) + (stats.health[level - 1] * (this.isRanged(true) ? 0.01125 : 0.015))) : 0
         const onHitWitsEnd = items.includes("3091") ? ([15, 15, 15, 15, 15, 15, 15, 15, 25, 35, 45, 55, 65, 75, 76.25, 77.5, 78.75, 80])[level - 1] : 0
         const onHitNashorsTooth = items.includes("3115") ? 15 + (stats.abilityPower[level - 1] * 0.2) : 0

         const physicalDamage = onHitGuinsoo + onHitTitanic + (items.includes("1043") ? 15 : 0)
         const magicDamage = onHitWitsEnd + onHitNashorsTooth
         return [physicalDamage, magicDamage]
      },
   },
   computed:{
      showModeTooltips(){  
         return this.$store.getters.getShowModeTooltips
      },
      mainStats(){
         return this.$store.getters.getCalculatedStats(true)
      },
      targetStats(){
         return this.$store.getters.getCalculatedStats(false)
      },
      mythics(){
         return this.$store.getters.getMythics
      },
      damageEqual(){ // damage at equal levels
         return this.mainStats.map(infoSet => {
            return Array.from({length: 18}, (_, level) => {
               return this.calculateDamage(infoSet.stats, infoSet.items, level + 1, level + 1)[this.strikeType].totalDamage
            })
         })
      },
      damageSelectedChanging(){ // damage selected level vs changing
         return this.mainStats.map(infoSet => {
            return Array.from({length: 18}, (_, level) => {
               return this.calculateDamage(infoSet.stats, infoSet.items, this.level(true), level + 1)[this.strikeType].totalDamage
            })
         })
      },
      damageChangingSelected(){ // damage changing level vs selected
         return this.mainStats.map(infoSet => {
            return Array.from({length: 18}, (_, level) => {
               return this.calculateDamage(infoSet.stats, infoSet.items, level + 1, this.level(false))[this.strikeType].totalDamage
            })
         })
      },
      datasetsToDisplay(){
         return this.mainStats.map((infoSet, index) => {
            return {
               label: infoSet.title,
               backgroundColor: `hsla(${this.graphColors[index]}, 100%, 65%, 1)`,
               borderColor: `hsla(${this.graphColors[index]}, 100%, 65%, 0.8)`,
               data: this[`damage${this.levelSetting}`][index]
            }
         })
      },
      watchables(){
         return [this.strikeType, this.levelSetting, ...this.damageEqual[0], ...this.damageChangingSelected[0], ...this.damageSelectedChanging[0]]
      },
   },
   watch: {
      watchables: {
         handler(){
            this.updateChart()
         },
         deep: true
      }
   }
}
</script>

<style scoped>
.resultsContainer{
   width: 100%;
   margin-bottom: 10em;
}
.resultsContainer > h2{
   padding-block: 1rem;
   text-align: center;
}
.chart{
   flex-direction: column;
}
.chartContainer{
   width: 100%;
}
.graphSettingsContainer{
   width: 100%;
   margin-bottom: 1em;
}
.modeButtonsContainer{
   width: 100%;
   position: relative;
   display: flex;
   flex-direction: row;
}
.modeButton{
   display: flex;
   width: calc(100% / 3);
   padding-inline: 1em;
   padding-block: 0.85em;
   font-size: 0.8rem;
   z-index: 1;
}
.modeButton:hover{
   background: var(--highlight1);
}
.modeButtonHighlight{
   position: absolute;
   width: calc(100% / 3);
   height: 100%;
   background: var(--highlight1);
   z-index: 0;
}
.modeButtonHighlight.averageStrike, .modeButtonHighlight.Equal{
   left: 0;
}
.modeButtonHighlight.nonCriticalStrike, .modeButtonHighlight.SelectedChanging{
   left: calc(100% / 3);
}
.modeButtonHighlight.criticalStrike, .modeButtonHighlight.ChangingSelected{
   left: calc(100% / 3 * 2);
}
.levelTooltip, .strikeTooltip{
   width: 16rem;
   margin-left: -8rem;
}
.levelTooltip{
   bottom: 220%;
}
.modeButtonsContainer:hover .levelTooltip, .modeButtonsContainer:hover .strikeTooltip{
   visibility: visible;
}
@media (min-width: 768px) {
   .chartContainer, .graphSettingsContainer{
      width: 48rem;
   }
   .modeButton{
      font-size: 1rem;
      padding-block: 0.7em;
   }
   .levelTooltip, .strikeTooltip{
      width: 20rem;
      margin-left: -10rem;
   }
}
</style>