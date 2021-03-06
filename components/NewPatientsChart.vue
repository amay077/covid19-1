<template>
  <data-view
    :title="displayTitle"
    title-id="new-patients-chart"
    :date="date"
    url="https://www.pref.aichi.jp/site/covid19-aichi/kansensya-kensa.html"
    :remarks="remarks"
  >
    <template v-if="showSelector" v-slot:button>
      <data-selector v-model="dataKind" :items="dataKinds" />
    </template>

    <div style="flex-grow: 1; display: flex; align-items: start;">
      <time-bar-line-chart
        chart-id="new-patients-chart"
        :chart-data="chartData"
        legend-order-kind="desc"
      />
    </div>

    <template v-slot:infoPanel>
      <data-view-basic-info-panel
        :l-text="displayInfo.lText"
        :s-text="displayInfo.sText"
        :unit="displayInfo.unit"
      />
    </template>
  </data-view>
</template>

<script lang="ts">
import { Component, Vue, Prop } from 'vue-property-decorator'
import dayjs, { Dayjs } from 'dayjs'
import * as Enumerable from 'linq'
import DataView from '@/components/DataView.vue'
import DataSelector, { SelectorItem } from '@/components/DataSelector.vue'
import DataViewBasicInfoPanel from '@/components/DataViewBasicInfoPanel.vue'
import DateSelectSlider from '@/components/DateSelectSlider.vue'
import TimeBarLineChart, { GraphData } from '@/components/TimeBarLineChart.vue'
import { PatientsSummaryDaily, PatientsSummaryWeekly } from '~/utils/types'

export type NewPatientsAverageType = {
  date: Dayjs
  count: number
  average7days: number | undefined
  count7days: number | undefined
}

export type NewSeniorPatientsAverageType = NewPatientsAverageType

type DataKind = 'daily-transition' | 'weekly-transition' | 'daily-cumulative'

type DisplayInfo = {
  lText: string
  sText: string
  unit: string
}

@Component({
  components: {
    DataView,
    DataSelector,
    DataViewBasicInfoPanel,
    DateSelectSlider,
    TimeBarLineChart
  }
})
export default class NewPatientsChart extends Vue {
  @Prop()
  public date?: string

  @Prop()
  public dailyData?: PatientsSummaryDaily[]

  @Prop()
  public weeklyData?: PatientsSummaryWeekly[]

  private readonly remarks = [
    '「新規感染者数」とは、愛知県が発表する「<a class=RemarksLink target=_blank href=https://www.pref.aichi.jp/site/covid19-aichi/kansensya-kensa.html>愛知県内発生事例</a>」を日別または週別に集計した人数です。(参考:<a class=RemarksLink target=_blank href=https://github.com/code4nagoya/covid19/blob/master/data/patients.csv>当サイトでCSV形式に加工したデータ</a>)。',
    '過去7日間の平均は、新規感染者数の後方7日移動平均値です'
  ]

  private readonly showSelector = true
  private dataKind: DataKind = 'daily-transition'
  private readonly dataKinds = [
    { key: 'weekly-transition', label: '週別' } as SelectorItem,
    { key: 'daily-transition', label: '日別' } as SelectorItem,
    { key: 'daily-cumulative', label: '累計' } as SelectorItem
  ]

  private readonly chartDataSet = new Map<DataKind, GraphData>()

  private get displayTitle(): string {
    return `${this.dataKind === 'daily-cumulative' ? '累計' : '新規'}感染者数${
      this.dataKind === 'weekly-transition' ? '(週別)' : ''
    }`
  }

  private formatDayBeforeRatio = (dayBeforeRatio: any) => {
    const dayBeforeRatioLocaleString = dayBeforeRatio.toLocaleString()
    const prefix = Math.sign(dayBeforeRatio) === 1 ? '+' : ''
    return `${prefix}${dayBeforeRatioLocaleString}`
  }

  private get displayDiffValue(): string {
    if (this.chartData.datasets[0].values.slice(-2)[0] === undefined) {
      return '-'
    }
    const lastDay = this.chartData.datasets[0].values.slice(-1)[0]
    const lastDayBefore = this.chartData.datasets[0].values.slice(-2)[0]
    return this.formatDayBeforeRatio(lastDay - lastDayBefore)
  }

  private get displayInfo(): DisplayInfo {
    if ((this.chartData?.datasets?.length ?? 0) <= 0) {
      return {
        lText: '-',
        sText: '-',
        unit: ''
      } as DisplayInfo
    }

    const dataset = this.chartData.datasets[0]
    const latestValueText = dataset.values.slice(-1)[0].toLocaleString()
    const diffLabel =
      this.dataKind === 'weekly-transition' ? '前週比' : '前日比'
    const latestDate = this.formatDateLabel(this.chartData.dates.slice(-1)[0])
    return {
      lText: latestValueText,
      sText: `${latestDate} 時点（${diffLabel}：${this.displayDiffValue} ${dataset.unit}）`,
      unit: dataset.unit
    }
  }

  private formatDateLabel = (date: string | string[]): string => {
    return (typeof date === 'string' ? [date] : date)
      .map(d => dayjs(d).format('M/D'))
      .join('～')
  }

  constructor() {
    super()

    this.chartDataSet.set(
      'daily-transition',
      this.buildDailyTransitionGraphData()
    )
    this.chartDataSet.set(
      'weekly-transition',
      this.buildWeeklyTransitionGraphData()
    )
    this.chartDataSet.set(
      'daily-cumulative',
      this.buildDailyCumulativeGraphData()
    )
  }

  private get chartData(): GraphData {
    const data = this.chartDataSet.get(this.dataKind)
    if (data) {
      return data
    } else {
      return {
        dates: [],
        datasets: []
      } as GraphData
    }
  }

  public static makeAverageNewPatients = (
    data: PatientsSummaryDaily[]
  ): Enumerable.IEnumerable<NewPatientsAverageType> => {
    const source = Enumerable.from(data).reverse()
    return source
      .select(d => d['日付'])
      .select((_, index) => source.skip(index).take(7))
      .select(d => {
        const first = d.first()
        const ave =
          d.count() === 7
            ? d
                .where(d => d['小計'] !== undefined)
                .average(d => Number(d['小計']))
            : undefined
        const cnt =
          d.count() === 7
            ? d.where(d => d['小計'] !== undefined).sum(d => Number(d['小計']))
            : undefined

        return {
          date: dayjs(dayjs(first['日付']).format('YYYY-MM-DD')), // 時刻を切り落とす
          count: Number(first['小計']),
          average7days: ave,
          count7days: cnt
        }
      })
      .reverse()
  }

  public static makeAverageSeniorsNewPatients = (
    data: PatientsSummaryDaily[]
  ): Enumerable.IEnumerable<NewSeniorPatientsAverageType> => {
    const source = Enumerable.from(data).reverse()

    const sumSexesNum = (sexes: { [key: string]: number }): number => {
      if (sexes == null) {
        return 0
      }

      let num = 0
      Object.keys(sexes).forEach(key => {
        const x = sexes[key]
        if (x == null || Number.isNaN(Number(x))) {
          return
        }
        num = num + Number(x)
      })

      return num
    }

    const getOver70sNum = (d: PatientsSummaryDaily) => {
      const y = d['年代']
      return (
        sumSexesNum(y['70代']) +
        sumSexesNum(y['80代']) +
        sumSexesNum(y['90代']) +
        sumSexesNum(y['100代']) +
        sumSexesNum(y['110代'])
      )
    }

    return source
      .select(d => d['日付'])
      .select((_, index) => source.skip(index).take(7))
      .select(d => {
        const first = d.first()
        const ave =
          d.count() === 7
            ? d
                .where(d => getOver70sNum(d) !== undefined)
                .average(d => Number(getOver70sNum(d)))
            : undefined
        const cnt =
          d.count() === 7
            ? d
                .where(d => getOver70sNum(d) !== undefined)
                .sum(d => Number(getOver70sNum(d)))
            : undefined

        return {
          date: dayjs(dayjs(first['日付']).format('YYYY-MM-DD')), // 時刻を切り落とす
          count: Number(getOver70sNum(first)),
          average7days: ave,
          count7days: cnt
        }
      })
      .reverse()
  }

  private buildDailyTransitionGraphData = (): GraphData => {
    const now = dayjs()
    const rows = NewPatientsChart.makeAverageNewPatients(this.dailyData ?? [])
      .where(d => d.date < now)
      .select(d => {
        return {
          date: d.date.format('YYYY-MM-DD'),
          count: d.count,
          average7days: d.average7days
        }
      })

    return {
      dates: rows.select(d => d.date).toArray(),
      datasets: [
        {
          type: 'bar',
          title: '陽性者数',
          unit: '人',
          values: rows.select(d => d.count).toArray(),
          order: 2
        },
        {
          type: 'line',
          title: '過去7日間の平均',
          unit: '人',
          values: rows.select(d => d.average7days).toArray(),
          order: 1
        }
      ]
    } as GraphData
  }

  private buildWeeklyTransitionGraphData = (): GraphData => {
    const now = dayjs()
    const rows = Enumerable.from(this.weeklyData ?? [])
      .where(d => dayjs(d['開始日']) < now)
      .select(d => {
        return {
          date: [
            dayjs(d['開始日']).format('YYYY-MM-DD'),
            dayjs(d['終了日']).format('YYYY-MM-DD')
          ],
          count: Number(d['小計'])
        }
      })

    return {
      dates: rows.select(d => d.date).toArray(),
      datasets: [
        {
          type: 'bar',
          title: '陽性者数',
          unit: '人',
          values: rows.select(d => d.count).toArray()
        }
      ]
    } as GraphData
  }

  private buildDailyCumulativeGraphData = (): GraphData => {
    let subTotal = 0
    const now = dayjs()
    const arr = (this.dailyData ?? [])
      .filter(d => dayjs(d['日付']) < now)
      .map(d => {
        subTotal += Number(d['小計'])
        return {
          date: dayjs(d['日付']).format('YYYY-MM-DD'),
          total: subTotal
        }
      })

    const rows = Enumerable.from(arr)

    return {
      dates: rows.select(d => d.date).toArray(),
      datasets: [
        {
          type: 'bar',
          title: '陽性者累計数',
          unit: '人',
          values: rows.select(d => d.total).toArray()
        }
      ]
    } as GraphData
  }
}
</script>
