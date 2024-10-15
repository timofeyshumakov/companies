<template>
  <div v-if="isLoading">Загрузка...</div>
  <main v-else class="main">
      <v-form @submit.prevent="handleSubmit">
      <v-select
        v-model="selectedUsers"
        :items="this.users"
        item-title="FULL_NAME"
        item-value="ID"
        label="Выберите пользователей"
        multiple
        chips
        clearable
      >
    </v-select>
    <v-select
      v-model="period"
      :items="[`Последние 30 дней`, `Квартал`, `Год`, `Диапазон месяцев`]"

      label="Выберите период"
      class="period"
    ></v-select>
      <v-btn type="submit" color="primary">Отправить</v-btn>
    </v-form>
    <v-data-table
        :headers="headers"
        :items="this.paginatedItems"
        :items-per-page="this.itemsPerPage"
        :pagination.sync="pagination"
        :server-items-length="items.length"
        sort-desc
        class="elevation-1"
    >
    <template v-slot:tfoot>
      <tfoot>
      <tr>
        <td v-for="obj in this.total" class="tfoot__td">{{obj}}</td>
      </tr>
    </tfoot>
      </template>
    </v-data-table>
    <v-row>
        <v-col>
          <v-select
            v-model="itemsPerPage"
            :items="[10, 25, 50]"
            label="Значений на странице"
            @update:modelValue="tabler"

          ></v-select>
        </v-col>
      </v-row>
      <v-pagination
        @input="fetchData"
        @change="fetchData"
        @click="fetchData"
        v-model="pagination.page"
        :length="pageCount"
        v-if="pageCount >= 2 && this.items.length > 0"
      ></v-pagination>
    <div v-bind:style="{ display: isBlock ? 'grid' : 'none' }" class="charts">
      <div v-for="n in 3" class="chart" :id="`chart${n}`"></div>
    </div>
  </main>
</template>

<script>
import Highcharts from 'highcharts';
import { ref } from 'vue';

export default {
  setup() {
    const isBlock = ref(false);

    const toggleDisplay = () => {
      isBlock.value = true;
    }
    return {
      isBlock,
      toggleDisplay,
    };
  },
  data() {
    return {
      monthsNames: [
        'Январь', 'Февраль', 'Март', 'Апрель', 'Май', 'Июнь',
        'Июль', 'Август', 'Сентябрь', 'Октябрь', 'Ноябрь', 'Декбрь'
      ],
      selectedDateRange: null,
      selectedUsers: [],
      total: [],
      paginatedItems: [],
      users: [],
      items: [],
      companies: [],
      period: "Последние 30 дней",
      previousPeriod:  "",
      itemsPerPage: 10,
      pagination: {
        page: 1,
        rowsPerPage: 10,
      },
      headers: [
        { title: 'Сотрудник', value: 'NAME', sortable: true },
        {
          title: 'Кол-во компаний, закреплённых за сотрудником',
          align: 'center',
          children: [
            { title: 'Предшествующий период', value: 'value1', align: 'center', sortable: true },
            { title: 'Текущий период', value: 'value2', align: 'center', sortable: true },
            { title: 'Динамика', value: 'value3', align: 'center', sortable: true },
          ],
        },
        {
          title: 'Кол-во сделок по закрепленным компаниям',
          align: 'center',
          children: [
            { title: 'Предшествующий период', value: 'value4', align: 'center', sortable: true },
            { title: 'Текущий период', value: 'value5', align: 'center', sortable: true },
            { title: 'Динамика', value: 'value6', align: 'center', sortable: true },
          ],
        },
        {
          title: 'Кол-во компаний без сделок',
          align: 'center',
          children: [
            { title: 'Предшествующий период', value: 'value7', align: 'center', sortable: true },
            { title: 'Текущий период', value: 'value8', align: 'center', sortable: true },
            { title: 'Динамика', value: 'value9', align: 'center', sortable: true },
          ],
        },
      ],
    };
  },
  computed: {
    pageCount() {
      return Math.ceil(this.selectedUsers.length / this.itemsPerPage);
    },
  },
  methods: {
    tabler(){
      if(this.paginatedItems.length < this.selectedUsers.length && this.paginatedItems.length < this.itemsPerPage) {
        this.handleSubmit();
      }else{
        this.paginationPages();
      }
    },
    paginationPages() {
      console.log(this.itemsPerPage);
      let start = (this.pagination.page - 1) * this.itemsPerPage;
      if(start >= this.selectedUsers.length){
        start = 0;
        this.pagination.page = 1;
      }
      const end = start + this.itemsPerPage;
      console.log(this.items);
      this.paginatedItems = this.items.slice(start, end);
    },
    fetchData(){
      this.handleSubmit();
    },
    getSelectedDataAttribute() {
      const selectedItem = this.selectedUsers[0].find(
        (item) => item.value === this.selectedValue
      );
      if (selectedItem) {
        return selectedItem['data-custom'];
      } else {
        return null;
      }
    },
    async handleSubmit(){
      if(this.previousPeriod !== this.period){ 
        this.items = [];
        this.pagination.page = 1;
      }

      this.previousPeriod = this.period;
      let i = 0;
      const maxTotal = 50;
      const sliceEnd = this.pagination.page * maxTotal  > this.selectedUsers.length ? this.selectedUsers.length : this.pagination.page * maxTotal;
      const sliceStart = (this.pagination.page - 1) * maxTotal;
      const months = {
        "Последние 30 дней": 1,
        "Квартал": 3,
        "Год": 12,
        "Диапазон месяцев": 1,
      }[this.period];
      const period = 2678400 * months;
      const now = Math.floor(Date.now() / 1000);
      const unixTimestamp = now - period * 2;
      const date = new Date(unixTimestamp * 1000);
      const findDate = date.toISOString().replace(".000Z", "");
      const ids = this.items.map(obj => obj.ID);
      const paginatedUsers = this.selectedUsers.slice(sliceStart, sliceEnd);
      const filtredUsers = paginatedUsers.filter(value => !ids.includes(value));
      if(filtredUsers.length === 0){
        this.paginationPages();
        return; 
      }
      const compainesData = await this.callApi('crm.company.list', { "ASSIGNED_BY_ID": filtredUsers, ">DATE_CREATE": findDate }, "");
      let assignedUsers = {previous: {}, current: {}, dynamic: {}};
      let assignedUsersCount = {previous: {}, current: {}, dynamic: {}};
      for(i = 0; i < compainesData.length; i++) {
        const needleDate = Math.floor(new Date(compainesData[i].DATE_CREATE).getTime()/ 1000);
        if(now - needleDate > period){
          if(!assignedUsers.previous[compainesData[i].ASSIGNED_BY_ID]){
            assignedUsers.previous[compainesData[i].ASSIGNED_BY_ID] = [];
          }
          assignedUsers.previous[compainesData[i].ASSIGNED_BY_ID].push(compainesData[i].ID);
        }else{
          if(!assignedUsers.current[compainesData[i].ASSIGNED_BY_ID]){
            assignedUsers.current[compainesData[i].ASSIGNED_BY_ID] = [];
          }
          assignedUsers.current[compainesData[i].ASSIGNED_BY_ID].push(compainesData[i].ID);
        }
      }

      let compainesIds = compainesData.map(object => +object.ID);
      const dealsData = compainesIds.length > 0 ? await this.callApi('crm.deal.list', { "COMPANY_ID": compainesIds, ">DATE_CREATE": findDate }, "") : [];
      let compainesWithoutDeals = {previous: [], current:[], dynamic: []};
      let totalCount = {previous: [], current:[], dynamic: []};
      const sortedDealsData = {previous: [], current: []};
      dealsData.forEach(deal => {
        if(now - new Date(deal.DATE_CREATE).getTime() / 1000 > period) {
          sortedDealsData.previous.push(deal);
        }else{
          sortedDealsData.current.push(deal);
        }
      });

      let totalCountSum = {previous: 0, current: 0, dynamic: 0};
      let compainesWithoutDealsSum = {previous: 0, current: 0, dynamic: 0};
      let assignedUsersSum = {previous: 0, current: 0, dynamic: 0};

      for(let u = 0; u < this.selectedUsers.length; u++){
        compainesWithoutDeals.previous[u] = 0;
        compainesWithoutDeals.current[u] = 0;
        totalCount.previous[u] = 0;
        totalCount.current[u] = 0;

        if(assignedUsers.previous[this.selectedUsers[u]]){
          for(i = 0; i < assignedUsers.previous[this.selectedUsers[u]].length; i++){
            const count = sortedDealsData.previous.filter(item => item.COMPANY_ID === assignedUsers.previous[this.selectedUsers[u]][i]).length;
            totalCount.previous[u] += count;
            if(count === 0){
              compainesWithoutDeals.previous[u] += 1;
            }
          }
        }else{
          assignedUsers.previous[this.selectedUsers[u]] = [];
        }

        if(assignedUsers.current[this.selectedUsers[u]]){
          for(i = 0; i < assignedUsers.current[this.selectedUsers[u]].length; i++){
            const count = sortedDealsData.current.filter(item => item.COMPANY_ID === assignedUsers.current[this.selectedUsers[u]][i]).length;
            totalCount.current[u] += count;
            if(count === 0){
              compainesWithoutDeals.current[u] += 1;
            }
          }
        }else{
          assignedUsers.current[this.selectedUsers[u]] = [];
        }

        compainesWithoutDeals.dynamic[u] = compainesWithoutDeals.current[u] - compainesWithoutDeals.previous[u];
        totalCount.dynamic[u] = totalCount.current[u] - totalCount.previous[u];
        assignedUsersCount.current[u] = assignedUsers.current[this.selectedUsers[u]].length;
        assignedUsersCount.previous[u] = assignedUsers.previous[this.selectedUsers[u]].length;
        assignedUsersCount.dynamic[u] =assignedUsersCount.current[u] - assignedUsersCount.previous[u];
      }

      for (const key in assignedUsers.current) {
        assignedUsersSum.current += assignedUsers.current[key].length;
      }
      for (const key in assignedUsers.previous) {
        assignedUsersSum.previous += assignedUsers.previous[key].length;
      }
      assignedUsersSum.dynamic = assignedUsersSum.current - assignedUsersSum.previous;

      for (const key in totalCount.current) {
        totalCountSum.current += totalCount.current[key];
      }
      for (const key in totalCount.previous) {
        totalCountSum.previous += totalCount.previous[key];
      }
      totalCountSum.dynamic = totalCountSum.current - totalCountSum.previous;

      for (const key in compainesWithoutDeals.current) {
        compainesWithoutDealsSum.current += compainesWithoutDeals.current[key];
      }
      for (const key in compainesWithoutDeals.previous) {
        compainesWithoutDealsSum.previous += compainesWithoutDeals.previous[key];
      }
      compainesWithoutDealsSum.dynamic = compainesWithoutDealsSum.current - compainesWithoutDealsSum.previous;

      assignedUsersCount.previous.total = assignedUsersSum.previous;

      let items = paginatedUsers.map((val, index ) => ({
        ID: val,
        NAME: this.users.find(obj => obj.ID  === val).FULL_NAME,
        value1: assignedUsersCount.previous[index],
        value2: assignedUsersCount.current[index],
        value3: assignedUsersCount.dynamic[index],
        value4: totalCount.previous[index],
        value5: totalCount.current[index],
        value6: totalCount.dynamic[index],
        value7: compainesWithoutDeals.previous[index],
        value8: compainesWithoutDeals.current[index],
        value9: compainesWithoutDeals.dynamic[index],
      }));

      for(i = 0; i < items.length; i++) {
        if(!this.items[sliceStart + i]){
          this.items[sliceStart + i] = items[i];
        }
      }
      

      if(this.total.length === 0){
        this.total = {
          ID: "Итого",
          value1: assignedUsersSum.previous,
          value2: assignedUsersSum.current,
          value3: assignedUsersSum.dynamic,
          value4: totalCountSum.previous,
          value5: totalCountSum.current,
          value6: totalCountSum.dynamic,
          value7: compainesWithoutDealsSum.previous,
          value8: compainesWithoutDealsSum.current,
          value9: compainesWithoutDealsSum.dynamic,
        };
      }else{
        this.total.value1 += assignedUsersSum.previous;
        this.total.value2 += assignedUsersSum.current;
        this.total.value3 += assignedUsersSum.dynamic;
        this.total.value4 += totalCountSum.previous;
        this.total.value5 += totalCountSum.current;
        this.total.value6 += totalCountSum.dynamic;
        this.total.value7 += compainesWithoutDealsSum.previous;
        this.total.value8 += compainesWithoutDealsSum.current;
        this.total.value9 += compainesWithoutDealsSum.dynamic;
      }

      this.chartBuild('Кол-во компаний, закреплённых за сотрудником, за период', 'chart1', [this.total.value1, this.total.value2, this.total.value3]);
      this.chartBuild('Кол-во созданных сделок по закрепленным компаниям за период', 'chart2', [this.total.value4, this.total.value5, this.total.value6]);
      this.chartBuild('Кол-во компаний, закреплённых за сотрудником, без сделок за период', 'chart3', [this.total.value7, this.total.value8, this.total.value9]);
      this.paginationPages();

    },
    async callApi(method, filter, select){

      if(!filter){
        filter = {};
      }
      let total = 0;
      let maxTotal = 50;
      let data = {};

      await new Promise((resolve, reject) => {
        BX24.callMethod(method, {
          filter: filter,
          select: select,
        },(res) => {
          if (res.data()) {
            total = res.total();
            data = res.data();
            resolve(data);
          }
        });
      });

    if(total > maxTotal){
        let cmd = {};
      const iterations = Math.ceil(total / maxTotal);
      for (let i = 0; i < iterations; i++) {
        const key = `cmd${i}`;
        const value = {
          method: method,
          params: {
            start: i * maxTotal,
            filter,
            select: select
          }
        };
        cmd[key] = value;
      }
      await new Promise((resolve, reject) => {
        BX24.callBatch(
          cmd, (res) => {
            let resultData = [];
            for( let i = 0; i < iterations; i++ ){
              let key = `cmd${i}`;
              data = res[key].data();
              resultData.push(data);
            }
            resultData = resultData.flat();
            data = resultData;
            resolve(resultData);
          });
        })
      }
      return data;
    },
    chartBuild(title, elementId, chartData){
      Highcharts.chart(elementId, {
        chart: {
          type: 'column'
        },
        title: {
          text: title
        },
        xAxis: {
          categories: ['предшествующий период', 'текущий период', 'динамика', 'предшествующий период', 'текущий период', 'динамика', 'предшествующий период', 'текущий период', 'динамика']
        },
        yAxis: {
          title: {
            text: ''
          },
          min: 0,
        },
        series: [{
          name: 'Данные',
          data: chartData
        }]
      });
      this.toggleDisplay();
    }
  },
  async mounted(){
    this.users = await this.callApi('user.get', "", "");
    this.users.forEach(user => {
      user.FULL_NAME = `${user.SECOND_NAME ? user.SECOND_NAME : ''} ${user.NAME ? user.NAME : ''} ${user.LAST_NAME ? user.LAST_NAME : ''}`;
    });
  }
}
</script>

<style lang="sass">

::-webkit-scrollbar
  background: white

::-webkit-scrollbar-thumb
  background: #e6e6e6
  border-radius: 1rem

::-webkit-scrollbar-thumb:active
  background: #e1e1e1

.main
  display: flex
  flex-direction: column
  gap: 0.75rem
  padding: 0.5rem

.period
  min-width: 12rem
  max-width: 12rem

.charts
  grid-template-columns: repeat(2, 1fr)
  gap: 0.75rem

.chart
  border: 1px solid black
  border-radius: 1rem

.header
  margin: 1.5rem 0
  display: flex
  gap: 1rem
  width: 100%
  justify-content: space-between
  align-items: center

.tfoot__td:not(:first-child)
  text-align: center

.v-data-table-footer
  display: none

.v-input__details
  display: none

.v-btn.v-btn--density-default
  height: 3rem

.v-form
  width: 100%
  display: flex
  align-items: flex-start
  gap: 0.75rem
</style>


