<script>
  import { scaleLinear, scaleBand, scaleTime } from 'd3-scale'
  import { covid } from './cases_vacc_census_20211120.js'
  import { hospitalisations } from './hospitalisations.js'
  import { Graphic, Section, RectangleLayer, PointLayer, Line, XAxis, YAxis } from '@snlab/florence'
  import DataContainer from '@snlab/florence-datacontainer'
  
  // overall settings
  const padding = { left: 80, bottom: 40, top: 5, right: 5 }
  const color = 'rgb(93, 134, 156)'
  
  // entire dataset
  const covid_data_raw = new DataContainer(covid)
  const hosp_data_raw = new DataContainer(hospitalisations)
  let covid_data = covid_data_raw
  let hosp_data = hosp_data_raw

  // data for facet 1
  const covid_per_region = covid_data
    .groupBy('region')
    .summarise({ total_count: { cases: 'sum' } })

  // data for facet 2
  $: covid_distribution = covid_data
    .bin({ column: 'vaccination_rate', method: 'EqualInterval', numClasses: 15 })
    .summarise({ total_count: { vaccination_rate: 'count' } })

  // data for facet 4
  $: hospOverTime = hosp_data
    .groupBy("DATE")
    .summarise({ total_hosp: { TOTAL_IN: "sum" } })
    .arrange({ DATE: "ascending" }) // sort by month (otherwise line will criss-cross)
    .mutate({ date_as_date: row => new Date(row.DATE) }); // convert to proper date so we can use date scale

  // data for select ui
  const defaultValue = '**All Regions**'
  let selectedRegion = defaultValue
  $: {
    if (selectedRegion === defaultValue) {
      covid_data = covid_data_raw
      hosp_data = hosp_data_raw
    } else {
      covid_data = covid_data_raw
        .filter(row => row.region === selectedRegion)
      hosp_data = hosp_data_raw
        .filter(row => row.REGION === selectedRegion)
    }
  }

  function doSomethingOnMouseOver(eventInfo) {
    console.log(eventInfo.key)
    if (selectedRegion === eventInfo.key) {
      selectedRegion = defaultValue
    } else {
      selectedRegion = eventInfo.key
    } 
  }

  function doSomethingOnMouseOut(eventInfo) {
    console.log(eventInfo.key)
    selectedRegion = defaultValue
  }
</script>

<div class="graph">
  <div class="main-chart">
    <!-- main chart -->
    <Graphic width={825} height={825}>
      <Section
        x1={0}
        x2={0.49}
        y1={0}
        y2={0.49}
        {padding}
        flipY
        scaleX={scaleLinear().domain(covid_per_region.domain('total_count')).nice()}
        scaleY={scaleBand().domain(['Wallonia', 'Brussels', 'Flanders']).padding(0.1)}
      >
        <!-- contents of 1st section -->
        <RectangleLayer 
          x1={covid_per_region.map('region', region => 0)}
          x2={covid_per_region.column('total_count')}
          y1={covid_per_region.column('region')}
          y2={({ scaleY }) => covid_per_region.map('region', region => scaleY(region) + scaleY.bandwidth())}
          fill={color}
          keys={covid_per_region.column('region')}
          onClick={doSomethingOnMouseOver}
        />
        <XAxis title="Total Count" tickCount = {5}/>
        <YAxis title="Region" />
      </Section>
      <Section
        x1={0.51}
        x2={1}
        y1={0}
        y2={0.49}
        {padding}
        flipY
        scaleX={scaleLinear().domain(covid_distribution.domain('bins'))}
        scaleY={scaleLinear().domain([0, covid_distribution.domain('total_count')[1]])}
      >
        <!-- contents of 2nd section -->
        <RectangleLayer 
          x1={covid_distribution.map('bins', bin => bin[0])}
          x2={covid_distribution.map('bins', bin => bin[1])}
          y1={covid_distribution.map('total_count', x => 0)}
          y2={covid_distribution.column('total_count')}
          fill={color}
        />
        <XAxis tickCount={5} labelFontSize={8} title="Vaccination Rate"/>
        <YAxis labelFontSize={8} title="Count"/>
      </Section>
       <Section
        x1={0}
        x2={0.49}
        y1={0.51}
        y2={1}
        {padding}
        flipY
        scaleX={scaleLinear().domain(covid_data_raw.domain('older_than_20_with_high_degree'))}
        scaleY={scaleLinear().domain(covid_data_raw.domain('vaccination_rate'))}
      >
        <PointLayer x={covid_data.column('older_than_20_with_high_degree')} y={covid_data.column('vaccination_rate')} opacity={0.6} fill={color} />
        <XAxis title="Percentage of adult population with higher education degree" />
        <YAxis title="Percentage of total population that is fully vaccinated" />
      </Section>
      <Section
        x1={0.51}
        x2={1}
        y1={0.51}
        y2={1}
        {padding}
        flipY
        scaleX={scaleTime().domain(hospOverTime.domain('date_as_date'))}
        scaleY={scaleLinear().domain(hospOverTime.domain('total_hosp'))}
      >
        <!-- contents of 4th section -->
        <Line
          x={hospOverTime.column('date_as_date')}
          y={hospOverTime.column('total_hosp')}
          stroke={color}
          strokeWidth={2}
        />
        <XAxis tickCount={6} title="Date"/>
        <YAxis title="Number of people in hospital"/>
      </Section>
    </Graphic>
  </div>
</div>

<style>

</style>