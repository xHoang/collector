<template>
   <div class="container">
      <div class="championContainer main" v-if="main !== undefined">
         <baseChampionInfo :target="main.id" :isMain="true" />
         <championILSelector :isMain="true" />
         <div class="itemsPreviewContainer" v-if="mainItems.length">
            <div class="itemPreview" v-for="itemName in mainItems" :key="itemName.id" :class="{passive: passives.find(passive => passive.id === itemName)}">
               <img :src="`http://ddragon.leagueoflegends.com/cdn/${patch}/img/item/${item(itemName).image.full}`" :title="item(itemName).name"/>
               <span v-if="passives.find(passive => passive.id === itemName)">{{passive(itemName).description}}</span>
            </div>
         </div>
         <calculatedChampionInfo :isMain="true" />
      </div>
      <div class="championContainer target" v-if="target !== undefined">
         <baseChampionInfo :target="target.id" :isMain="false" />
         <championILSelector :isMain="false" />
         <div class="itemsPreviewContainer" v-if="targetItems.length">
            <div class="itemPreview" v-for="itemName in targetItems" :key="itemName.id" :class="{passive: passives.find(passive => passive.id === itemName)}">
               <img :src="`http://ddragon.leagueoflegends.com/cdn/${patch}/img/item/${item(itemName).image.full}`" :title="item(itemName).name"/>
               <span v-if="passives.find(passive => passive.id === itemName)">{{passive(itemName).description}}</span>
            </div>
         </div>
         <calculatedChampionInfo :isMain="false" />
      </div>
   </div>
</template>

<script>
import baseChampionInfo from '@/components/baseChampionInfo.vue'; import championILSelector from '@/components/championILSelector.vue'; import calculatedChampionInfo from '@/components/calculatedChampionInfo.vue'
export default {
   name: 'championComparison',
   components: {
      baseChampionInfo, championILSelector, calculatedChampionInfo
   },
   data(){
      return{
         hoveredItem: undefined,
      }
   },
   methods:{
      item(name){
         return this.$store.getters.getItem(name)
      },
      passive(item){
         return this.passives.find(passive => passive.id === item)
      },
   },
   computed:{
      patch(){
         return this.$store.getters.getPatch
      },
      main(){
         return this.$store.getters.getMainChampion
      },
      target(){
         return this.$store.getters.getTargetChampion
      },
      mainItems(){
         return this.$store.getters.getSelectedItems(true)
      },
      targetItems(){
         return this.$store.getters.getSelectedItems(false)
      },
      passives(){
         return this.$store.getters.getPassives
      }
   }
}
</script>

<style scoped>
.container{
   display: flex;
   flex-direction: row;
}
.championContainer{
   flex: 1 0 50%;
   max-width: 50%;
}
.main{
   background: hsla(120, 100%, 50%, 0.035);
}
.target{
   background: hsla(0, 100%, 50%, 0.035);
}
.itemsPreviewContainer{
   display: flex;
   justify-content: center;
   flex-wrap: wrap;
   padding-block: 1em;
}
.itemPreview{
   position: relative;
}
.itemPreview.passive::after{
   content: 'P';
   font-size: 12px;
   font-weight: bold;
   justify-content: center;
   align-items: center;
   display: flex;
   width: 1.2em;
   height: 1.2em;
   color: var(--main1);
   border-radius: 0 0 0 4px;
   background: var(--bg1);
   right: 0;
   top: 0;
   position: absolute;
}
.itemPreview span{
   width: 140px;
   margin-left: -70px;
}
.itemPreview:hover span{
   visibility: visible;
}
</style>