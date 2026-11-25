<template>
  <svg
    style="
      position: absolute;
      width: 1px;
      height: 1px;
      opacity: 0;
      overflow: hidden;
      pointer-events: none;
    "
    xmlns="http://www.w3.org/2000/svg"
  >
    <defs>
      <pattern
        v-for="value in RETROFIT_VALUES"
        :id="`${value}-hatch`"
        width="3"
        height="3"
        patternUnits="userSpaceOnUse"
        patternTransform="rotate(45)"
      >
        <rect width="3" height="3" :fill="retrofitStyle[value].color" />
        <line x1="0" y1="0" x2="0" y2="12" stroke="yellow" stroke-width="2" />
      </pattern>
      <pattern
        id="global-hatch"
        width="3"
        height="3"
        patternUnits="userSpaceOnUse"
        patternTransform="rotate(45)"
      >
        <rect width="3" height="3" fill="#00000000" />
        <line x1="0" y1="0" x2="0" y2="12" stroke="yellow" stroke-width="2" />
      </pattern>
    </defs>
  </svg>
  <div class="fit row">
    <div class="col-auto" style="overflow-y: scroll; height: 100%; width: 28%">
      <q-expansion-item
        v-model="showComputedFeatures"
        icon="science"
        label="Building Retrofit Computations"
        caption="Choose retrofit packages and predict impact"
      >
        <q-form @submit="predict" @reset="">
          <div class="q-px-sm q-ma-sm">
            <q-space />
            <div class="text-subtitle1 q-pt-xs text-weight-medium"
              >Selection</div
            >

            <div class="bg-blue-grey-1" style="min-height: 64px">
              <q-chip
                size="sm"
                removable
                class="text-white"
                :color="
                  buildingIdToVisualize == building ? 'positive' : 'primary'
                "
                @remove="
                  selectBuildings(
                    selectedBuildingIds.filter(v => v != building)
                  )
                "
                @click="
                  highlightBuilding(building);
                  buildingIdToVisualize = building;
                "
                v-for="building in selectedBuildingIds"
                >{{ building }}</q-chip
              >
            </div>
            <div class="text-subtitle1 q-pt-md text-weight-medium"
              >Retrofit Information Selected Buildings</div
            >
            <q-table
              dense
              flat
              hide-bottom
              :columns="
                ['name', ...RETROFIT_VALUES].map(r => ({
                  name: r,
                  label: retrofitStyle[r]?.label.split(' ')[0] ?? '',
                  format: (val, row) =>
                    `${isNumber(row[r]) ? row[r]?.toFixed(2) : row[r]}` + ` `,
                  headerStyle: 'width: 8px',
                  autoWidth: false
                }))
              "
              :rows="[
                {
                  name: beforeRetrofitData.name,
                  ...beforeRetrofitData.selected
                },
                {
                  name: afterRetrofitData.name,
                  ...afterRetrofitData.selected
                }
              ]"
              row-key="name"
            />
            <div
              class="text-subtitle1 q-pt-md text-weight-medium"
              style="line-height: 1.1"
              >Building Visualization</div
            >
            <div class="text-caption text-grey-7 q-pb-sm">
              Archetype:
              {{
                buildingIdToVisualize
                  ? (archetypeMap[
                      buildingProperties.get(buildingIdToVisualize)?.archetype
                    ] ?? "Unknown")
                  : "---"
              }}
            </div>
            <div class="bg-blue-grey-1">
              <q-img
                :src="`api/idf/?bag_id=${buildingIdToVisualize}&user_id=${user_uuid}`"
                style="margin: auto"
                width="128"
                height="128"
                :ratio="16 / 10"
              >
                <!-- <q-skeleton
                  style="margin: auto"
                  size="256px"
                  square
                  animation="fade"
              /> -->
              </q-img>
            </div>

            <q-separator></q-separator>
            <div class="text-subtitle1 q-pt-md text-weight-medium"
              >Choose Passive Retrofit Package</div
            >
            <q-option-group dense :options="retrofits" v-model="retrofit">
              <template #label-0="opt">
                <div class="row items-center">
                  <span>{{ opt.label }} (Default)</span>
                </div>
              </template>
              <template #label="opt">
                <div class="row items-center">
                  <span
                    :title="
                      JSON.stringify(opt.value?.values)
                        .substring(1)
                        .replace('}', '')
                    "
                    >{{ opt.label }}</span
                  >
                </div>
              </template>
            </q-option-group>

            <!-- <q-select
            clearable
            v-model="retrofit"
            :options="retrofits"
            label="Choose Passive Retrofit Package"
          >
            <template #option="scope">
              <q-item v-bind="scope.itemProps">
                <q-item-section>
                  <q-item-label :title="scope.opt.value">{{
                    scope.opt.label
                  }}</q-item-label>
                  <q-item-label caption>{{
                    JSON.stringify(scope.opt.value)
                      .substring(1)
                      .replace("}", "")
                  }}</q-item-label>
                </q-item-section>
              </q-item>
            </template>
          </q-select> -->
            <div class="text-subtitle1 q-pt-md text-weight-medium"
              >Choose Heating Retrofit</div
            >
            <q-option-group
              :options="activeHeatingRetrofits"
              type="radio"
              v-model="heatingRetrofit"
              dense
            >
              <template #label="opt">
                <div class="row items-center">
                  <span
                    >{{ opt.label }}
                    {{
                      opt.value.name ==
                      activeHeatingRetrofits[defaultHeatingRetrofitIndex]?.value
                        .name
                        ? "(Default)"
                        : ""
                    }}</span
                  >
                </div>
              </template>
            </q-option-group>

            <div class="text-subtitle1 q-pt-md text-weight-medium"
              >Choose Cooling Retrofit</div
            >
            <q-option-group
              :options="activeCoolingRetrofits"
              type="radio"
              v-model="coolingRetrofit"
              dense
            >
              <template #label="opt">
                <div class="row items-center">
                  <span
                    >{{ opt.label }}
                    {{
                      opt.value.name ==
                      activeCoolingRetrofits[defaultCoolingRetrofitIndex]?.value
                        .name
                        ? "(Default)"
                        : ""
                    }}</span
                  >
                </div>
              </template></q-option-group
            >
            <!-- <q-radio
            v-for="retro in activeRetrofits.filter(
              r => r.value.type == 'cooling'
            )"
            v-model="retro.value.enabled"
            >{{ retro.label }}</q-radio
          > -->
            <div class="q-gutter-md q-py-md">
              <q-btn
                class="q-mt-md"
                label="Apply Retrofit"
                type="submit"
                color="primary"
              />

              <q-btn
                class="q-mt-md"
                label="Reset All Retrofits"
                @click="reset"
                color="negative"
                type="button"
              />
            </div>
          </div>
        </q-form>
      </q-expansion-item>
      <q-expansion-item
        v-model="showSocialFeatures"
        icon="map"
        label="Social Feature Overlays"
        caption="Activate the social feature overlays"
      >
        <div class="q-px-sm q-ma-sm">
          <div class="text-subtitle1 q-pt-xs text-weight-medium">Usage</div>
          <q-option-group
            v-model="socialFeature"
            :options="socialFeaturesUsage"
            color="primary"
            type="radio"
            dense
          />

          <div class="text-subtitle1 q-pt-md text-weight-medium">Tenure</div>
          <q-option-group
            v-model="socialFeature"
            :options="socialFeaturesTenure"
            color="primary"
            type="radio"
            dense
          />

          <div class="text-subtitle1 q-pt-md text-weight-medium"
            >Demographics</div
          >
          <q-option-group
            v-model="socialFeature"
            :options="socialFeaturesDemographic"
            color="primary"
            type="radio"
            dense
          />

          <div class="text-subtitle1 q-pt-md text-weight-medium"
            >Households</div
          >
          <q-option-group
            v-model="socialFeature"
            :options="socialFeaturesHousehold"
            color="primary"
            type="radio"
            dense
          />
          <div class="text-subtitle1 q-pt-md text-weight-medium">Financial</div>
          <q-option-group
            v-model="socialFeature"
            :options="socialFeaturesFinancial"
            color="primary"
            type="radio"
            dense
          />

          <div
            v-if="socialFeature"
            class="text-subtitle1 q-pt-md text-weight-medium"
            >Overall Distribution</div
          >

          <Responsive v-if="socialFeature" class="w-full">
            <template #main="{ width }">
              <Chart
                :size="{ width, height: 100 }"
                :data="hist"
                :axis="{
                  primary: {
                    type: 'linear',
                    domain: [dataLow, dataHigh],
                    trigger: selectedSquareFeatureValue // NOTE: Workaround for making the VerticalMarker reactive.
                  },
                  secondary: {
                    type: 'linear',
                    ticks: 3
                  }
                }"
                :margin="{ left: 10, right: 20, top: 10, bottom: 10 }"
              >
                <template #layers>
                  <Area
                    :dataKeys="['x', 'y']"
                    type="monotone"
                    :areaStyle="{ fill: '#1976d2', stroke: '#1976d2' }"
                  />
                  <!-- TODO: Cap the square indicator by datalow/datahigh -->
                  <Marker
                    v-if="selectedSquareFeatureValue >= dataLow"
                    :value="
                      Math.min(
                        Math.max(selectedSquareFeatureValue, dataLow),
                        dataHigh
                      )
                    "
                    label="Square"
                    color="#e76f51"
                    :strokeWidth="3"
                    :strokeDasharray="0"
                    axis="primary"
                  />
                </template>
              </Chart>
            </template>
          </Responsive>
          <span
            v-if="socialFeature"
            style="
              width: 100%;
              position: relative;
              text-align: center;
              display: inline-block;
              top: -15px;
            "
            >{{ socialLabel }}</span
          >
          <span v-if="socialFeature">
            <i>
              Upon selecting a square, the red vertical line indicates how that
              square relates to the overall distribution for the social feature.
              If no line appears, the selected square has no data.
            </i>
          </span>
        </div>
      </q-expansion-item>
    </div>
    <div class="col-auto" style="width: 72%">
      <mgl-map
        :map-style="style"
        :center="[4.462456, 51.926517]"
        :zoom="zoom"
        :attribution-control="false"
        @map:zoom="fadeSquare"
      >
        <mgl-custom-control
          v-if="showComputedFeatures"
          class="maplibregl-ctrl maplibregl-ctrl-group translucent"
          :position="Position.TOP_LEFT"
        >
          <div style="display: flex">
            <q-card style="width: 224px; height: 104px">
              <!--
    <q-card-section horizontal>
      <q-card-section class="q-pt-xs">
        <div class="text-overline">Overline</div>
        <div class="text-h5 q-mt-sm q-mb-xs">Title</div>
        <div class="text-caption text-grey">
          Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do
          eiusmod tempor incididunt ut labore et dolore magna aliqua.
        </div>
      </q-card-section>

      <q-card-section class="col-5 flex flex-center">
        <q-img
          class="rounded-borders"
          src="https://cdn.quasar.dev/img/parallax2.jpg"
        />
      </q-card-section>
    </q-card-section>

    <q-separator />

    <q-card-actions>
      <q-btn flat round icon="event" />
      <q-btn flat> 7:30PM </q-btn>
      <q-btn flat color="primary"> Reserve </q-btn>
    </q-card-actions> -->

              <q-card-section horizontal class="">
                <q-card-section class="q-pr-none">
                  <q-icon name="paid" color="green" size="5em"></q-icon>
                </q-card-section>
                <q-card-section class="q-pl-sm">
                  <div class="text-h6">Credits</div>
                  <span style="display: flex">
                    <div class="text-h4">{{ credits }}</div>
                    <q-separator vertical class="q-ma-sm"></q-separator>
                    <div
                      class="text-h6"
                      :style="`color: ${retrofitCost > credits ? 'red' : 'green'}`"
                      >{{ retrofitCost < 0 ? "+" : ""
                      }}{{ retrofitCost != 0 ? -retrofitCost : "" }}</div
                    ></span
                  ><q-space></q-space>
                  <q-item>
                    <q-item-section class="text-h4"></q-item-section>

                    <q-item-section side class="text-h5"></q-item-section>
                  </q-item>
                </q-card-section>
              </q-card-section>
            </q-card>
            <!-- ..
            <q-space></q-space>
            <q-card style="width: 192px; height: 96px"></q-card> -->
          </div>
        </mgl-custom-control>

        <mgl-custom-control
          class="maplibregl-ctrl maplibregl-ctrl-group translucent"
          :position="Position.BOTTOM_LEFT"
        >
          <div style="display: flex">
            <div style="padding: 4px; pointer-events: auto">
              <div
                class="text-subtitle1 text-weight-bold text-center"
                style="user-select: none"
                >Retrofit effect for selection</div
              >
              <Responsive>
                <template #main="{ height }">
                  <Chart
                    :key="beforeRetrofitData.selected.heat"
                    :size="{
                      width: 300,
                      height: Math.min(1000, windowHeight - 250)
                    }"
                    :data="[
                      {
                        name: beforeRetrofitData.name,
                        ...beforeRetrofitData.selected
                      },
                      {
                        name: afterRetrofitData.name,
                        ...afterRetrofitData.selected
                      }
                      // transformRetrofitResultForChart(afterRetrofitData)
                    ]"
                    :margin="{
                      left: 0,
                      top: 20,
                      right: 20,
                      bottom: 0
                    }"
                    :axis="{
                      primary: {
                        type: 'band'
                      },
                      secondary: {
                        domain: [0, Math.max(maxRetroValue + 20, 400)],
                        type: 'linear',
                        ticks: 8
                      }
                    }"
                  >
                    <template #layers>
                      <Grid strokeDasharray="2,2" />
                      <Bar
                        v-for="rk in RETROFIT_VALUES"
                        :dataKeys="['name', rk]"
                        :barStyle="{ fill: retrofitStyle[rk].color }"
                      />
                    </template>

                    <!-- <template #widgets>
                      <Tooltip
                        color="black"
                        :hide-line="true"
                        :format="'_>10.2f'"
                        :config="{
                          name: { color: 'black', label: '' },
                          heat: { color: '#5b5a5a', label: 'Heating Demand' },
                          cool: { color: '#5b5a5a', label: 'Cooling Demand' },
                          elec: {
                            color: '#5b5a5a',
                            label: 'Electricity Usage (kWh/year)'
                          },
                          gas: {
                            color: '#5b5a5a',
                            label: 'Gas usage (m³/year)'
                          },
                          co2: { color: '#5b5a5a', label: 'CO² Emission' }
                        }"
                      />
                    </template> -->
                  </Chart>
                </template>
              </Responsive>
            </div>
            <div style="display: flex; flex-direction: column" class="q-mr-md">
              <q-space></q-space>
              <span
                style="display: flex"
                class="vertical-middle q-pa-xs"
                v-for="value in RETROFIT_VALUES"
              >
                <svg width="16px" height="16px" class="q-mx-xs">
                  <rect
                    width="16px"
                    height="16px"
                    :fill="retrofitStyle[value].color"
                  ></rect>
                </svg>
                {{ retrofitStyle[value].label }}
                {{
                  retrofitStyle[value].unit
                    ? `(${retrofitStyle[value].unit})`
                    : ""
                }}
              </span>
              <!-- <q-separator />
              <q-separator />

              <span style="display: flex" class="vertical-middle q-pa-xs"
                ><svg width="16px" height="16px" class="q-mx-xs">
                  <rect
                    width="16px"
                    height="16px"
                    fill="url(#global-hatch)"
                  ></rect>
                </svg>
                From Selected Buildings
              </span> -->
              <span style="display: flex" class="vertical-middle q-pa-xs"
                ><svg width="16px" height="16px" class="q-mx-xs"></svg>
              </span>
            </div>
            <div style="padding: 4px; display: flex; flex-direction: column">
              <q-space></q-space>
              <div
                class="text-subtitle1 text-weight-bold text-center"
                style="user-select: none; width: 200px"
                >Underheating/Overheating hours for selection</div
              >
              <Responsive>
                <template #main="{ height }">
                  <Chart
                    :key="beforeRetrofitData.selected.under"
                    :size="{
                      width: 200,
                      height: Math.min(250, windowHeight - 250)
                    }"
                    :data="[
                      {
                        name: beforeRetrofitData.name,
                        ...beforeRetrofitData.selected
                      },
                      {
                        name: afterRetrofitData.name,
                        ...afterRetrofitData.selected
                      }
                      // transformRetrofitResultForChart(afterRetrofitData)
                    ]"
                    :margin="{
                      left: 0,
                      top: 20,
                      right: 20,
                      bottom: 0
                    }"
                    :axis="{
                      primary: {
                        type: 'band'
                      },
                      secondary: {
                        domain: [0, Math.max(maxRetroValueExtra + 20, 30000)],
                        type: 'linear',
                        ticks: 8
                      }
                    }"
                  >
                    <template #layers>
                      <Grid strokeDasharray="2,2" />
                      <Bar
                        v-for="rk in RETROFIT_VALUES_EXTRA"
                        :dataKeys="['name', rk]"
                        :barStyle="{ fill: retrofitStyle[rk].color }"
                      />
                      <Labels
                        v-for="rk in RETROFIT_VALUES_EXTRA"
                        :dataKeys="['name', rk]"
                        :labelStyle="{ fill: retrofitStyle[rk].color }"
                      />
                    </template>

                    <!-- <template #widgets>
                      <Tooltip
                        color="black"
                        :hide-line="true"
                        :format="'_>10.2f'"
                        :config="{
                          name: { color: 'black', label: '' },
                          heat: { color: '#5b5a5a', label: 'Heating Demand' },
                          cool: { color: '#5b5a5a', label: 'Cooling Demand' },
                          elec: {
                            color: '#5b5a5a',
                            label: 'Electricity Usage (kWh/year)'
                          },
                          gas: {
                            color: '#5b5a5a',
                            label: 'Gas usage (m³/year)'
                          },
                          co2: { color: '#5b5a5a', label: 'CO² Emission' }
                        }"
                      />
                    </template> -->
                  </Chart>
                </template>
              </Responsive>
            </div>
            <div style="display: flex; flex-direction: column" class="q-mr-md">
              <q-space></q-space>
              <span
                style="display: flex"
                class="vertical-middle q-pa-xs"
                v-for="value in RETROFIT_VALUES_EXTRA"
              >
                <svg width="16px" height="16px" class="q-mx-xs">
                  <rect
                    width="16px"
                    height="16px"
                    :fill="retrofitStyle[value].color"
                  ></rect>
                </svg>
                {{ retrofitStyle[value].label }}
                {{
                  retrofitStyle[value].unit
                    ? `(${retrofitStyle[value].unit})`
                    : ""
                }}
              </span>
              <span style="display: flex" class="vertical-middle q-pa-xs"
                ><svg width="16px" height="16px" class="q-mx-xs"></svg>
              </span>
            </div>
          </div>
        </mgl-custom-control>

        <mgl-custom-control
          class="maplibregl-ctrl maplibregl-ctrl-group translucent"
          :position="Position.BOTTOM_RIGHT"
          v-if="!socialFeature"
        >
          <div style="margin: 8px">
            <div style="text-align: right; width: 150px"
              >Building Heating Demand (kWh/m²)</div
            >

            <table align="right">
              <tr
                ><td align="right">{{
                  paintBuildings["fill-extrusion-color"].findLast(() => true)[5]
                }}</td>
                <td align="right" rowspan="8">
                  <div
                    :style="`height:128px;width:16px;background: linear-gradient(0deg,${paintBuildings['fill-extrusion-color'].findLast(() => true)[4]} 0%, ${paintBuildings['fill-extrusion-color'].findLast(() => true)[6]} 100%);`"
                  ></div
                ></td>
              </tr>
              <tr><td>.</td></tr>
              <tr><td>.</td></tr>
              <tr><td>.</td></tr>
              <tr
                ><td align="right">{{
                  paintBuildings["fill-extrusion-color"].findLast(() => true)[3]
                }}</td></tr
              >
            </table>
          </div>
        </mgl-custom-control>

        <mgl-custom-control
          class="maplibregl-ctrl maplibregl-ctrl-group translucent"
          :position="Position.BOTTOM_RIGHT"
          v-if="socialFeature"
        >
          <div style="margin: 8px">
            <div style="text-align: right; width: 150px">{{ socialLabel }}</div>

            <table align="right">
              <tr
                ><td align="right">{{
                  fillColor?.["fill-color"]?.[5]?.toLocaleString("en-us", {
                    maximumFractionDigits: 2,
                    useGrouping: false
                  })
                }}</td>
                <td align="right" rowspan="8">
                  <div
                    :style="`height:128px;width:16px;background: linear-gradient(0deg,${socialFeature?.lowColor} 0%,${socialFeature?.midColor ? socialFeature?.midColor + ' 50%, ' : ''} ${socialFeature?.highColor} 100%);`"
                  ></div
                ></td>
              </tr>
              <tr><td>.</td></tr>
              <tr><td>.</td></tr>
              <tr><td>.</td></tr>
              <tr
                ><td align="right">{{
                  fillColor?.["fill-color"]?.[3]?.toLocaleString("en-us", {
                    maximumFractionDigits: 2,
                    useGrouping: false
                  })
                }}</td></tr
              >
            </table>
          </div>
        </mgl-custom-control>

        <mgl-custom-control :position="Position.TOP_RIGHT">
          <button
            title="Pan and interact"
            class="maplibregl-ctrl-icon"
            :style="{
              color: currentTool === 'general' ? 'deepSkyBlue' : 'grey'
            }"
            @click="currentTool = 'general'"
          >
            <q-icon name="pan_tool_alt" size="sm" />
            <!-- <q-icon ></q-icon>   -->
            <!-- <svg>
              <path :d="mdiCursorDefault" fill="currentColor" />
            </svg> -->
          </button>
        </mgl-custom-control>

        <mgl-custom-control :position="Position.TOP_RIGHT">
          <button
            title="Select multiple buildings"
            class="maplibregl-ctrl-icon"
            :style="{
              color: currentTool === 'select' ? 'deepSkyBlue' : 'grey'
            }"
            @click="currentTool = 'select'"
          >
            <q-icon name="highlight_alt" size="sm" />

            <!-- <svg>
              <path :d="mdiSelect" fill="currentColor" />
            </svg> -->
          </button>
        </mgl-custom-control>

        <!-- <mgl-marker
          :coordinates="[4.125239818838533, 51.9785307340928]"
        ></mgl-marker>
        <mgl-marker
          :coordinates="[4.457889979857661, 51.871067129276618]"
        ></mgl-marker> -->
        <mgl-popup
          v-if="popupShow"
          :close-button="false"
          :coordinates="popupCoordinates"
          :close-on-click="false"
          :close-on-move="false"
          :text="popupText"
        />
        <mgl-geo-json-source source-id="buildings" :data="buildings">
          <mgl-fill-extrusion-layer
            layer-id="buildings"
            :paint="paintBuildings"
            @mousemove="hoverBuildingEnter"
            @mouseout="hoverBuildingLeave"
            @click="clickBuilding"
          >
          </mgl-fill-extrusion-layer>
        </mgl-geo-json-source>

        <mgl-geo-json-source
          source-id="geojson"
          :generate-id="true"
          :data="rotterdamUrl"
        >
          <mgl-fill-layer
            layer-id="geojson"
            :paint="{ ...defaultPaint, ...fillColor }"
            @mousemove="hoverSquareEnter"
            @mouseout="hoverSquareLeave"
            @click="squareZoom"
          />

          <mgl-line-layer layer-id="select" :paint="paintSelection" />
        </mgl-geo-json-source>

        <mgl-geo-json-source source-id="boundaries" :data="boundaryUrl">
          <mgl-line-layer
            layer-id="boundaries"
            :paint="paintBoundaries"
          ></mgl-line-layer>
        </mgl-geo-json-source>
        <mgl-geo-json-source
          source-id="boundariesLabels"
          :data="boundariesJSON"
        >
          <mgl-symbol-layer
            layer-id="names"
            :layout="{ 'text-field': ['get', 'buurtnaam'] }"
            :paint="{
              'text-color': '#9e0000',
              'text-opacity': 0.75,
              'text-halo-color': 'white',
              'text-halo-width': 1
            }"
          ></mgl-symbol-layer>
        </mgl-geo-json-source>

        <mgl-navigation-control /> </mgl-map
    ></div>
  </div>
</template>

<script setup lang="ts">
import {
  MglMap,
  MglNavigationControl,
  MglCustomControl,
  MglGeoJsonSource,
  MglFillLayer,
  MglLineLayer,
  MglSymbolLayer,
  MglFillExtrusionLayer,
  MglPopup,
  MglMarker,
  Position,
  useMap
} from "@indoorequal/vue-maplibre-gl";

import {
  Responsive,
  Bar,
  Chart,
  Line,
  Marker,
  Area,
  Grid,
  Tooltip,
  HoverBar,
  Group
} from "vue3-charts";

import workerUrl from "maplibre-gl/dist/maplibre-gl-worker.mjs?worker&url";
import {
  computed,
  nextTick,
  onMounted,
  ref,
  shallowRef,
  toRaw,
  useTemplateRef,
  watch
} from "vue";
import { colors, Color, useQuasar } from "quasar";
import { center, featureCollection, isNumber, union } from "@turf/turf";
import { loadPyodide, PyodideAPI } from "pyodide";
import { setWorkerUrl } from "maplibre-gl";
import { v4 as uuidv4 } from "uuid";
import * as np from "numpy-ts";

import bdata from "../assets/kralingseveer_buildings.json";
import {
  mdiCursorDefault,
  mdiCursorDefaultClick,
  mdiCursorMove,
  mdiPan,
  mdiSelect
} from "@mdi/js";
import VerticalMarker from "@/components/VerticalMarker.vue";
import Labels from "@/components/LabelsLayer.vue";
setWorkerUrl(workerUrl);
// async function main() {
const { lighten } = colors;
// console.log("Pyodide and GeoPandas are ready!");
// return pyodide;
// }

import { useWindowSize } from "@vueuse/core";

const { height: windowHeight } = useWindowSize();

const rotterdamUrl = "roffa_grid_gwrf_combined_wgs84_fixed.geojson";
const boundaryUrl = "roffa_boundaries_wgs84_geoman_edit.geojson";
// style="position: absolute; width: 1px; height: 1px; opacity: 0; overflow: hidden; pointer-events: none;"

const RETROFIT_VALUES = ["heat", "cool", "elec", "gas", "co2"] as const;
type RetrofitValue = (typeof RETROFIT_VALUES)[number];

const RETROFIT_VALUES_EXTRA = ["over", "under"] as const;
type RetrofitValueExtra = (typeof RETROFIT_VALUES_EXTRA)[number];

type RetrofitValues = Record<RetrofitValue | RetrofitValueExtra, number>;
type RetrofitStyle = Record<
  RetrofitValue | RetrofitValueExtra,
  { label: string; color: string; unit?: string }
>;

const archetypeMap: Record<number, string> = {
  1: "Detached House",
  2: "Corner House",
  3: "Terraced House",
  4: "Ground Floor Apartment",
  5: "Upstairs Apartment"
};

const retrofitStyle: RetrofitStyle = {
  ...(Object.fromEntries(
    RETROFIT_VALUES.map(c => [
      c,
      { color: "#000000", label: "Some Retrofit Value" }
    ])
  ) as RetrofitStyle),
  heat: { label: "Heating Demand", color: "#ff4800", unit: "kWh/m²" },
  cool: { label: "Cooling Demand", color: "#0096c7", unit: "kWh/m²" },
  elec: { label: "Electricity Usage", color: "#b1b050", unit: "kWh/m²" },
  gas: { label: "Gas Usage", color: "#389b0a", unit: "kWh/m²" },
  co2: { label: "CO2 Emission", color: "#4f4f4e", unit: "kgCO2/m²" },
  over: { label: "Overheating Hours", color: "#ff7272", unit: "hour" },
  under: { label: "Underheating Hours", color: "#6877ff", unit: "hour" }
};

// const initRetrofit = () =>
//   Object.fromEntries(RETROFIT_VALUES.map(c => [c, 0])) as RetrofitValues;

interface RetrofitResult {
  name: string;
  selected: RetrofitValues;
  total: RetrofitValues;
}

// const beforeRetrofitData = ref<RetrofitResult>({
//   name: "Before Retrofit",
//   selected: initRetrofit(),
//   total: initRetrofit()
// });

const style = "minimal.json";
const zoom = 12;
const mapHolder = useMap();
const buildings = ref<{ [key: string]: object }>({});
const buildingProperties = computed(
  () =>
    new Map<number, object>(
      buildings.value?.features?.map(b => [parseInt(b["id"]), b.properties]) ??
        []
    )
);
const creditAllowance = 1000;
const $q = useQuasar();
const pyodideInstance = ref<PyodideAPI>();
const showSocialFeatures = ref<boolean>(false);
const showComputedFeatures = ref<boolean>(false);

type ActiveRetrofitName =
  | "retrofit_no_heating"
  | "retrofit_no_cooling"
  | "retrofit_dx_airco"
  | "retrofit_gas_boiler"
  | "retrofit_heat_pump"
  | "retrofit_district_heating";

type PassiveRetrofitName = "shallow" | "deep" | "deep_limited_inf";

const retrofitCostMap: Record<
  ActiveRetrofitName | PassiveRetrofitName,
  number
> = {
  retrofit_no_cooling: 0,
  retrofit_no_heating: 0,
  retrofit_dx_airco: 8,
  retrofit_gas_boiler: 0,
  retrofit_heat_pump: 12,
  retrofit_district_heating: 3,
  shallow: 35,
  deep: 40,
  deep_limited_inf: 45
};
// intersected.loc[idx, "total_floor_area"] = building.n_total_floor_area
// intersected.loc[idx, "ground_floor_area"] = building.bag_ground_floor_area
// intersected.loc[idx, "pp_ht_floor"] = building.pp_ht_floor
// intersected.loc[idx, "compact_ratio"] = building.pp_compactness_ratio
// intersected.loc[idx, "adiab_wall_area"] = building.bag_area_walls_adiab
// intersected.loc[idx, "archetype"] = building.n_archetype
interface BuildingSanityCheck {
  violated: (building: any) => boolean;
  message: string;
}

const buildingSanityChecks: BuildingSanityCheck[] = [
  {
    violated: (building: any) => building?.total_floor_area > 500,
    message: "Very large total floor area"
  },
  {
    violated: (building: any) => building?.ground_floor_area > 500,
    message: "Very large building footprint"
  },
  {
    violated: (building: any) => building?.pp_ht_floor > 4,
    message: "Floor height is very tall"
  },
  {
    violated: (building: any) => building?.pp_ht_floor < 2,
    message: "Floor height is very short"
  },
  {
    violated: (building: any) => building?.compact_ratio > 5,
    message: "Very high compactness ratio"
  },
  {
    violated: (building: any) =>
      building?.archetype == 1 && building?.adiab_wall_area > 0,
    message: "Detached building with adiabatic walls"
  }
];

const buildingRetrofitCost = function (building: object): number {
  let cost = 0;
  if (building.retrofit.passive.name) {
    console.log(
      building.retrofit.passive.name,
      retrofitCostMap?.[building.retrofit.passive.name]
    );
    cost += retrofitCostMap?.[building.retrofit.passive.name] ?? 0;
  }
  for (const act of building.retrofit?.active ?? []) {
    console.log(building.retrofit.active, retrofitCostMap?.[act.name]);
    cost += retrofitCostMap?.[act.name] ?? 0;
  }
  return cost;
};

const credits = computed<number>(() => {
  let acc = 0;
  for (const building of Object.values(retrofittedBuildings.value)) {
    acc += buildingRetrofitCost(building);
  }
  return creditAllowance - acc;
});

const retrofitCost = computed<number>(() => {
  let cost = 0;
  const modRetro = selectedBuildingIds.value.filter(
    id => id in retrofittedBuildings.value
  );
  const newRetro = selectedBuildingIds.value.filter(
    id => !(id in retrofittedBuildings.value)
  );

  const newRetroCost = ((retrofitCostMap?.[coolingRetrofit.value.name] ?? 0) +
    (retrofitCostMap?.[heatingRetrofit.value.name] ?? 0) +
    (retrofitCostMap?.[retrofit.value.name] ?? 0)) as number;

  cost += (newRetro.length + modRetro.length) * newRetroCost;

  modRetro.forEach(
    mr => (cost -= buildingRetrofitCost(retrofittedBuildings.value[mr]))
  );

  return cost;
});

const activeCoolingRetrofits = ref<ActiveRetrofit[]>([
  {
    label: "None",
    value: { name: "retrofit_no_cooling", retro_type: "cooling" }
  },
  {
    label: "Air Conditioning",
    value: {
      name: "retrofit_dx_airco",
      retro_type: "cooling",
      params: { dx_cop: 4 }
    }
  }
]);
const activeHeatingRetrofits = ref<ActiveRetrofit[]>([
  {
    label: "None",
    value: { name: "retrofit_no_heating", retro_type: "heating" }
  },
  {
    label: "Gas Boiler",
    value: {
      name: "retrofit_gas_boiler",
      retro_type: "heating",
      params: { boiler_eff: 0.95 }
    }
  },
  {
    label: "Heat Pump",
    value: {
      name: "retrofit_heat_pump",
      retro_type: "heating",
      params: { hp_cop: 5 }
    }
  }
  // {
  //   label: "District Heating",
  //   value: { name: "retrofit_district_heating", retro_type: "heating" }
  // }
]);
const defaultCoolingRetrofitIndex = activeCoolingRetrofits.value.findIndex(
  r => r.value.name === "retrofit_no_cooling"
);
const defaultHeatingRetrofitIndex = activeHeatingRetrofits.value.findIndex(
  r => r.value.name === "retrofit_gas_boiler"
);
const coolingRetrofit = ref<ActiveRetrofit>(
  activeCoolingRetrofits.value[defaultCoolingRetrofitIndex]?.value
);
const heatingRetrofit = ref<ActiveRetrofit>(
  activeHeatingRetrofits.value[defaultHeatingRetrofitIndex]?.value
);

interface ActiveRetrofit {
  label: string;
  value: {
    name: ActiveRetrofitName | null;
    retro_type: "heating" | "cooling";
    params?: {
      [key: string]: number;
    };
  };
}

interface RetrofitPackage {
  label: string;
  value: {
    name: PassiveRetrofitName | null;
    values: {
      wall?: number;
      roof?: number;
      floor?: number;
      window?: number;
      inf_rate?: number;
    };
  };
}
// For each of the retrofit package they choose, they should be able to see the results with
// all heating and cooling system types.
// For heating we have gas boiler
// ((1)gas_boiler, boiler_eff:0.95, (2)heat_pump, hp_cop: 5 and (3) district heating(district_heating)).
// For cooling we have air conditioning (dx_airco, dx_cop:4).
// Then we should also show overheating hours in case that there is no cooling system (no_cooling), and underheating in case there is no heating (no_heating).

const retrofits = ref<RetrofitPackage[]>([
  {
    label: "None",
    value: { name: null, values: {} }
  },
  {
    label: "Shallow",
    value: {
      name: "shallow",
      values: {
        wall: 3.9,
        roof: 4.2,
        floor: 3.1,
        window: 2
      }
    }
  },
  // {
  //   label: "Deep",
  //   value: {
  //     name: "deep",
  //     values: {
  //       wall: 6.2,
  //       roof: 8.3,
  //       floor: 5.7,
  //       window: 3
  //     }
  //   }
  // },
  {
    label: "Deep",
    value: {
      name: "deep_limited_inf",
      values: {
        wall: 3.9,
        roof: 4.2,
        floor: 3.1,
        window: 2,
        inf_rate: 0.0004
      }
    }
  }
]);
const retrofit = ref<RetrofitPackage>(retrofits.value[0]?.value);

const user_uuid = $q.cookies.useCookie("user", {
  default: () => "",
  expires: "30d"
});

// TODO: Nicest would be to have a tiny tool interface with some predefined tools
//       but I don't think Typescript supports case classes..
const currentTool = ref<"general" | "select">("general");
watch(currentTool, async newTool => {
  console.log("changed!", newTool);
  switch (newTool) {
    case "general":
      mapHolder?.map?.dragPan.enable();
      mapHolder?.map?.getCanvas().style.setProperty("cursor", "");
      break;
    case "select":
      mapHolder?.map?.dragPan.disable();
      mapHolder?.map?.getCanvas().style.setProperty("cursor", "crosshair");
      break;
  }
  console.log(currentTool.value);
});

watch(showSocialFeatures, async newVal => {
  if (!newVal) {
    socialFeature.value = undefined;
  }
});

let hoveredStateId: number | undefined = undefined;
const selectedStateId = ref<number>();
let hoveredBuildingStateId: number | undefined = undefined;
const selectedBuildingIds = ref<number[]>([]);
const buildingIdToVisualize = ref<number>();
const boundariesJSON = ref<object>({ data: {} });
const socialJSON = ref<object>();
const popupCoordinates = ref<[number, number]>([0, 0]);
const popupShow = ref(false);
const popupText = ref("");
let popupTimeout: number = 0;

// watch(buildingIdToVisualize, async newVal => {
//   const res = await fetch(
//     `api/idf/?user_id=${user_uuid.value}?building_id=${newVal}`,
//     {
//       method: "GET"
//     }
//   );
// });

watch(selectedBuildingIds.value, newVal => {
  if (newVal) {
    showComputedFeatures.value = true;
  }
});

interface SocialFeature {
  label: string;
  value: {
    name: string;
    lowColor: string;
    midColor?: string;
    highColor: string;
  };
}

const transformRetrofitResultForChart = function (res: RetrofitResult) {
  return {
    name: res.name,
    ...res.selected,
    ...Object.fromEntries(
      Object.entries(res.total).map(([key, value]) => [
        `t_${key}`,
        value - res.selected[key as RetrofitValue]
      ])
    )
  };
};

const socialFeaturesUsage: SocialFeature[] = [
  {
    label: "Average Gas Usage (m³/year)",
    value: {
      name: "avg_gas_consumption_mu",
      lowColor: "#edf941",
      midColor: "#ff5e00",
      highColor: "#da0000"
    }
  },
  {
    label: "Average Electricity Usage (kWh/year)",
    value: {
      name: "avg_electricity_consumption_mu",
      lowColor: "#fcfdff",
      midColor: "#3b9dee",
      highColor: "#001953"
    }
  }
];

const socialFeaturesTenure: SocialFeature[] = [
  {
    label: "Rental (%)",
    value: {
      name: "pct_rental_dwellings_mu",
      lowColor: "#fcfdff",
      highColor: "#3c0096"
    }
  },
  {
    label: "Social Rent (% of Rental)",
    value: {
      name: "pct_social_rent_mu",
      lowColor: "#fcfdff",
      midColor: "#ff8a30",
      highColor: "#832700"
    }
  },
  {
    label: "Home Owner (%)",
    value: {
      name: "pct_owner_occupied_dwellings_mu",
      lowColor: "#fcfdff",
      midColor: "#c44e99",
      highColor: "#a6034f"
    }
  }
];

const socialFeaturesFinancial: SocialFeature[] = [
  {
    label: "Income Band",
    value: {
      name: "income_band",
      lowColor: "#0c153e",
      midColor: "#696c51",
      highColor: "#fed401"
    }
  },
  {
    label: "Social Benefits (%)",
    value: {
      name: "pct_on_benefits_mu",
      lowColor: "#100e2d",
      highColor: "#29ffb8"
    }
  }
];

const socialFeaturesHousehold: SocialFeature[] = [
  {
    label: "Apartment Density (%)",
    value: {
      name: "pct_multi_family_dwellings_mu",
      lowColor: "#3a3a29",
      highColor: "#6f7bff"
    }
  },
  {
    label: "Average Household Size (Count)",
    value: {
      name: "avg_household_size_mu",
      lowColor: "#f7f7f3",
      midColor: "#5e9dbd",
      highColor: "#0a3972"
    }
  }
];

const socialFeaturesDemographic: SocialFeature[] = [
  {
    label: "Working Age (%)",
    value: {
      name: "pct_age_25_65_mu",
      lowColor: "#fcfdff",
      midColor: "#7cd874",
      highColor: "#0a5403"
    }
  },

  {
    label: "Elderly (%)",
    value: {
      name: "pct_age_65_plus_mu",
      lowColor: "#f2ff7d",
      midColor: "#1696a9",
      highColor: "#0834ba"
    }
  }
] as const;

const socialFeature = ref<SocialFeature["value"]>();

const retrofittedBuildings = ref<{ [key: string]: object }>({});

const socialLabel = computed(
  () =>
    [
      ...socialFeaturesDemographic,
      ...socialFeaturesFinancial,
      ...socialFeaturesHousehold,
      ...socialFeaturesTenure,
      ...socialFeaturesUsage
    ].find(s => s.value.name == socialFeature.value?.name)?.label
);

// NOTE: Might not work for categorical data.
const fillColor = computed(() => {
  if (socialFeature.value) {
    return {
      "fill-color": [
        "interpolate",
        ["linear"],
        ["get", socialFeature.value?.name], // Numerical GeoJSON property
        np.percentile(np.array(stats.value), 5),
        socialFeature.value?.lowColor,
        ...(socialFeature.value?.midColor
          ? [
              np.percentile(np.array(stats.value), 50),
              socialFeature.value?.midColor
            ]
          : []),
        np.percentile(np.array(stats.value), 95),
        socialFeature.value?.highColor
      ]
    };
  } else {
    return {};
  }
});

const selectedSquareFeatureValue = computed(() => {
  if (socialFeature.value && selectedStateId.value) {
    return (
      mapHolder?.map
        ?.querySourceFeatures("geojson")
        .find(f => f.id == selectedStateId.value)?.properties[
        socialFeature.value?.name
      ] ?? undefined
    );
  } else {
    return undefined;
  }
});

const stats = computed(() => {
  if (socialFeature.value) {
    return socialJSON.value.features
      .map(f => f.properties[socialFeature.value?.name])
      .filter(v => v != null && v > 0);
  } else {
    return [];
  }
});

const dataHigh = computed(() => hist.value?.[hist.value?.length - 1]["x"] ?? 0);
const dataLow = computed(() => hist.value?.[0]["x"] ?? 0);

// const dataHigh = computed(() => np.max(np.array(stats.value)));
// const dataLow = computed(() => np.min(np.array(stats.value)));

const hist = computed(() => {
  const arr = np.array(stats.value);
  const un = np.unique(arr);
  console.log(":D", un.size);
  const bins = un.size > 15 ? 10 : un.size - 2;
  const [counts, binEdges] = np.histogram(np.array(stats.value), bins);
  console.log(binEdges.toArray(), np.min(un), np.max(un));
  // np.random.seed(42);
  // const data = np.random.normal(0, 1, [1000]);

  // // Automatic binning (10 bins by default)
  // const [counts, binEdges] = np.histogram(
  //   np.array([1, 2, 3, 4, 5, 2, 3, 4, 1, 3])
  // );
  // console.log("Counts:", counts, "\n");
  // console.log("Bin edges:", binEdges, "\n");
  // console.log("Number of bins:", counts.shape[0], "\n");
  const edges = binEdges.toArray();
  const carr = counts.toArray();
  const ret = [carr[0], ...carr].map((c, i) => {
    return { x: Math.round(edges[i]), y: c };
  });
  console.log("RET", ret);
  return ret;
});

const defaultPaint = {
  "fill-outline-color": [
    "case",
    ["boolean", ["feature-state", "selected"], false],
    "#ff0000ff",
    "#a09c9c45"
  ],
  "fill-opacity": [
    "case",
    ["boolean", ["feature-state", "hover"], false],
    1,
    0.75
  ],
  "fill-color": "#7f837f94"
};

// let paint = ref({
//   "fill-color": [
//     "interpolate",
//     ["linear"],
//     ["get", "avg_gas_consumption_mu"], // Numerical GeoJSON property
//     0,
//     "#20cf2c",
//     // 500, '#3978eb',
//     2000,
//     "#6e004f"
//   ],
//   "fill-opacity": [
//     "case",
//     ["boolean", ["feature-state", "hover"], false],
//     1,
//     0.9
//   ]
// });

const paintBoundaries = {
  "line-color": "#fc0000",
  "line-layer-opacity": 0.5,
  "line-width": 2
};

const paintSelection = {
  "line-color": [
    "case",
    ["boolean", ["feature-state", "hover"], false],
    "#fc0000",
    ["boolean", ["feature-state", "retrofit"], false],
    "#249ef6",
    "#fc0000"
  ],
  "line-opacity": [
    "case",
    ["boolean", ["feature-state", "retrofit"], false],
    0.8,
    ["boolean", ["feature-state", "selected"], false],
    0.5,
    ["boolean", ["feature-state", "hover"], false],
    0.2,
    0
  ],
  "line-width": [
    "case",
    ["boolean", ["feature-state", "retrofit"], false],
    3,
    ["boolean", ["feature-state", "selected"], false],
    4,
    2
  ]
};

let paintBuildings = ref({
  // "fill-extrusion-outline-color": "#0000FF",

  "fill-extrusion-color": [
    "case",
    ["boolean", ["feature-state", "highlight"], false],
    "#ff0000",
    [
      "all",
      ["boolean", ["feature-state", "click"], false],
      ["boolean", ["feature-state", "retrofit"], false]
    ],
    "#26ea8e",
    ["boolean", ["feature-state", "click"], false],
    "#f8ff2d",
    ["boolean", ["feature-state", "retrofit"], false],
    "#3b77af",

    ["boolean", ["feature-state", "hover"], false],
    "#9c9c9c",
    [
      "interpolate",
      ["linear"],
      ["get", "init_heat"], // Numerical GeoJSON property
      0,
      "#31ff49",
      400,
      "#ff6200"
    ]
  ]
});

function message(str: string, color = "purple") {
  $q.notify({
    message: str,
    color: color,
    position: "top-right"
  });
}

async function clickBuilding(e) {
  console.log(e);
  selectBuildings([e.features[0].id]);
  e.preventDefault();
}

// NOTE: May need debouncing for IDF querying. Method call that starts late?
async function selectBuildings(buildingIds: number[]) {
  for (const oldSelectedBuildingId of selectedBuildingIds.value.splice(
    0,
    selectedBuildingIds.value.length
  )) {
    mapHolder?.map?.setFeatureState(
      { source: "buildings", id: oldSelectedBuildingId },
      { click: false }
    );
  }

  for (const newSelectedBuildingId of buildingIds) {
    selectedBuildingIds.value.push(newSelectedBuildingId);

    mapHolder?.map?.setFeatureState(
      { source: "buildings", id: newSelectedBuildingId },
      { click: true }
    );
  }

  buildingSanityChecks.forEach(check => {
    selectedBuildingIds.value.forEach(buildingId => {
      console.log(buildingProperties.value.get(buildingId));

      if (check.violated(buildingProperties.value.get(buildingId))) {
        message(
          `Sanity check failed for building '${buildingId}': ${check.message}, this may affect predictions.`,
          "orange"
        );
        highlightBuilding(buildingId, 3000);
      }
    });
  });

  buildingIdToVisualize.value = selectedBuildingIds.value[0];
}

function highlightBuilding(building: number, duration: number = 1000) {
  mapHolder?.map?.setFeatureState(
    { source: "buildings", id: building },
    { highlight: true }
  );
  resetHighlight(building, duration);
}

function resetHighlight(building: number, duration: number) {
  setTimeout(() => {
    mapHolder?.map?.setFeatureState(
      { source: "buildings", id: building },
      { highlight: false }
    );
  }, duration);
}

async function reset() {
  $q.dialog({
    message: "Are you sure you want to reset all retrofits?",
    ok: "Proceed",
    cancel: true
  }).onOk(async () => {
    const res = await fetch(`api/reset/?user_id=${user_uuid.value}`, {
      method: "GET",
      headers: {
        "Content-Type": "application/json"
      }
    });
    for (const [building_id, building] of Object.entries(
      retrofittedBuildings.value
    )) {
      mapHolder?.map?.setFeatureState(
        { source: "buildings", id: parseInt(building_id) },
        { retrofit: false }
      );
    }

    // Just to be sure, reset for all squares.
    mapHolder?.map?.querySourceFeatures("geojson").forEach(f => {
      mapHolder?.map?.setFeatureState(
        { source: "geojson", id: f.id },
        { retrofit: false }
      );
    });

    retrofittedBuildings.value = {};
    if (selectedStateId.value != undefined) {
      await loadBuildings(selectedStateId.value);
    }

    message("All retrofits have been reset!", "grey");
  });
}

async function predict() {
  if (selectedBuildingIds.value.length) {
    if (credits.value - retrofitCost.value < 0) {
      message(`Current retrofit plan exceeds credit limit!`, "orange");
      return;
    }

    if (
      retrofit.value.name === null &&
      coolingRetrofit.value.name ===
        activeCoolingRetrofits.value[defaultCoolingRetrofitIndex]?.value.name &&
      heatingRetrofit.value.name ===
        activeHeatingRetrofits.value[defaultHeatingRetrofitIndex]?.value.name
    ) {
      message(
        `Default retrofits selected, resetting retrofits for buildings: ${selectedBuildingIds.value}`,
        "grey"
      );
    } else {
      message(
        `Computing retrofit impact for buildings: ${selectedBuildingIds.value}`
      );
    }

    const res = await fetch(`api/predict/?user_id=${user_uuid.value}`, {
      method: "POST",
      body: JSON.stringify({
        buildings: selectedBuildingIds.value,
        r_package: retrofit.value,
        r_active: [coolingRetrofit.value, heatingRetrofit.value].filter(
          r =>
            // The backend interprets this as a no-op, if it is not sent.
            r?.name !=
              activeHeatingRetrofits.value[defaultHeatingRetrofitIndex]?.value
                .name &&
            r?.name !=
              activeCoolingRetrofits.value[defaultCoolingRetrofitIndex]?.value
                .name
        ),
        square_id: selectedStateId.value
      }),
      headers: {
        "Content-Type": "application/json"
      }
    });
    const json = JSON.parse(await res.json());
    for (const buildingId of selectedBuildingIds.value) {
      const prediction = json[buildingId];

      if (
        buildingId in retrofittedBuildings.value &&
        !("retrofit" in prediction)
      ) {
        delete retrofittedBuildings.value[buildingId.toString()];
        mapHolder?.map?.setFeatureState(
          { source: "buildings", id: buildingId },
          { retrofit: false }
        );
      } else if ("retrofit" in prediction) {
        retrofittedBuildings.value[buildingId.toString()] = prediction;
        mapHolder?.map?.setFeatureState(
          { source: "buildings", id: buildingId },
          { retrofit: true }
        );
      }
    }
    console.log("predres", json);

    await loadBuildings(selectedStateId.value);
    message(
      `Computation successfully finished for: ${selectedBuildingIds.value}`,
      "green"
    );
  } else {
    message(`Please select at least one building!`, "orange");
  }
}

function hoverBuildingEnter(e) {
  if (e.features.length > 0) {
    if (hoveredBuildingStateId) {
      mapHolder?.map?.setFeatureState(
        { source: "buildings", id: hoveredBuildingStateId },
        { hover: false }
      );
      popupShow.value = false;
      clearTimeout(popupTimeout);
    }
    hoveredBuildingStateId = e.features[0].id;

    mapHolder?.map?.setFeatureState(
      { source: "buildings", id: hoveredBuildingStateId },
      { hover: true }
    );

    popupCoordinates.value = e.lngLat;
    if (e.features[0].properties.has_retrofit) {
      clearTimeout(popupTimeout);
      popupText.value = `Retrofit Details: ${e.features[0].properties.retrofit_details}`; // TODO: Beautify the popup.
      popupTimeout = setTimeout(() => {
        if (hoveredBuildingStateId) {
          popupShow.value = true;
        }
      }, 500);
    }
  }
}

function hoverBuildingLeave(e) {
  if (hoveredBuildingStateId) {
    mapHolder?.map?.setFeatureState(
      { source: "buildings", id: hoveredBuildingStateId },
      { hover: false }
    );
  }
  hoveredBuildingStateId = undefined;
  popupShow.value = false;
  clearTimeout(popupTimeout);
}

function hoverSquareEnter(e) {
  if (e.features.length > 0) {
    if (hoveredStateId) {
      mapHolder?.map?.getCanvas().style.setProperty("cursor", "pointer");
      mapHolder?.map?.setFeatureState(
        { source: "geojson", id: hoveredStateId },
        { hover: false }
      );
    }
    if (
      selectedStateId.value === e.features[0].id &&
      mapHolder.map.getZoom() > 17
    )
      return;

    hoveredStateId = e.features[0].id;
    mapHolder?.map?.setFeatureState(
      { source: "geojson", id: hoveredStateId },
      { hover: true }
    );
  }
}

function hoverSquareLeave(e) {
  if (hoveredStateId) {
    mapHolder?.map?.getCanvas().style.setProperty("cursor", "");
    mapHolder?.map?.setFeatureState(
      { source: "geojson", id: hoveredStateId },
      { hover: false }
    );
  }
  hoveredStateId = undefined;
}

function fadeSquare(e) {
  mapHolder?.map?.setPaintProperty(
    "geojson",
    "fill-layer-opacity",
    (0.9 * Math.pow(zoom, 4)) / Math.pow(mapHolder.map.getZoom(), 4)
  );
}

const getDenormalizedRetrofitValue = function (
  iterable: IteratorObject<number> | number[],
  attribute: string
): number {
  const numer = np.nansum(
    np.array(
      iterable
        .map(id => buildingProperties.value.get(id))
        .map(b => (b?.[attribute] ?? 0) * (b?.total_floor_area ?? 0))
    ) ?? 0
  );
  const denom = np.nansum(
    np.array(
      iterable
        .map(id => buildingProperties.value.get(id))
        .map(b => b?.total_floor_area ?? 0)
    ) ?? 0
  );
  return numer / denom;
};

const beforeRetrofitData = computed<RetrofitResult>(() => {
  return {
    name: "Before",
    selected: Object.fromEntries(
      [...RETROFIT_VALUES, ...RETROFIT_VALUES_EXTRA].map(attr => [
        attr,
        getDenormalizedRetrofitValue(selectedBuildingIds.value, `init_${attr}`)
      ])
    ) as RetrofitValues,
    total: Object.fromEntries(
      [...RETROFIT_VALUES, ...RETROFIT_VALUES_EXTRA].map(attr => [
        attr,
        getDenormalizedRetrofitValue(
          buildingProperties.value.keys(),
          `init_${attr}`
        )
      ])
    ) as RetrofitValues
  };
});

const afterRetrofitData = computed<RetrofitResult>(() => {
  return {
    name: "After",
    selected: Object.fromEntries(
      [...RETROFIT_VALUES, ...RETROFIT_VALUES_EXTRA].map(attr => [
        attr,
        getDenormalizedRetrofitValue(selectedBuildingIds.value, `pred_${attr}`)
      ])
    ) as RetrofitValues,
    total: Object.fromEntries(
      [...RETROFIT_VALUES, ...RETROFIT_VALUES_EXTRA].map(attr => [
        attr,
        getDenormalizedRetrofitValue(
          buildingProperties.value.keys(),
          `pred_${attr}`
        )
      ])
    ) as RetrofitValues
  };
});

const maxRetroValue = computed(() =>
  Math.max(
    ...RETROFIT_VALUES.map(r => beforeRetrofitData.value.selected[r]),
    ...RETROFIT_VALUES.map(r => afterRetrofitData.value.selected[r])
  )
);

const maxRetroValueExtra = computed(() =>
  Math.max(
    ...RETROFIT_VALUES_EXTRA.map(r => beforeRetrofitData.value.selected[r]),
    ...RETROFIT_VALUES_EXTRA.map(r => afterRetrofitData.value.selected[r])
  )
);

//   const res = {} as RetrofitResult;
//     for (const attr of RETROFIT_VALUES) {
//         res.total[attr] = np.nansum(
//           np.array(
//             buildings.value.features.map(feat => feat.properties[`init_${attr}`])
//           )
//         ) as number;
//         beforeRetrofitData.value.selected[attr] = np.nansum(
//           np.array(
//             buildings.value.features
//               .filter(feat =>
//                 selectedBuildingIds.value.includes(parseInt(feat["id"]))
//               )
//               .map(feat => feat.properties[`init_${attr}`])
//           )
//         ) as number;
//         // afterRetrofitData.value[attr] = np.nanmean(
//         //   np.array(
//         //     buildings.value.features.map(feat => feat.properties[`pred_${attr}`])
//         //   )
//         // ) as number;
//       }
// }

async function loadBuildings(squareId: number) {
  // NOTE: Maplibre cuts polygons up when zooming in, which is why we have to do this annoying dance of
  //       recombining them based on the clicked id.
  const matchingFragments = mapHolder?.map?.querySourceFeatures("geojson", {
    sourceLayer: "geojson",
    filter: ["==", ["id"], squareId]
  });
  const combined = union(
    featureCollection([...matchingFragments, ...matchingFragments])
  );
  const square = combined.geometry.coordinates[0];

  mapHolder?.map?.flyTo({
    center: center(combined).geometry.coordinates,
    zoom: 18
  });

  // NOTE: This is for when we actually will use a backend.
  const res2 = await fetch(`api/buildings/?user_id=${user_uuid.value}`, {
    method: "POST",
    body: JSON.stringify({ coordinates: square, square_id: squareId }),
    headers: {
      "Content-Type": "application/json"
    }
  });

  // NOTE: This is for github demo-ing purposes.
  // const str = JSON.stringify({ coordinates: square });
  // const value = await pyodideInstance.value?.runPythonAsync(`
  //   square = json.loads('${str}')
  //   indices = gdf.sindex.query(convex_hull(Polygon(square["coordinates"])), predicate="intersects")
  //   intersected: gpd.GeoDataFrame = gdf.iloc[indices]
  //   intersected.to_json()
  // `);
  // const parsed = JSON.parse(value);

  // console.log(parsed);
  // buildings.value = parsed;
  buildings.value = JSON.parse(await res2.json());
  console.log("Buildings!", buildings.value);
  // console.log(
  //   "???",
  //   beforeRetrofitData.value.selected,
  //   beforeRetrofitData.value.total
  // );

  buildings.value.features.forEach(building => {
    if (building.properties.has_retrofit) {
      mapHolder?.map?.setFeatureState(
        { source: "buildings", id: parseInt(building.id) },
        { retrofit: true }
      );
      mapHolder?.map?.setFeatureState(
        { source: "geojson", id: squareId },
        { retrofit: true }
      );
    }
  });
}

async function squareZoom(e) {
  if (e._defaultPrevented) return;

  if (e.features) {
    if (
      selectedStateId.value === e.features[0].id &&
      mapHolder.map.getZoom() > 17
    )
      return;

    if (selectedStateId.value) {
      mapHolder?.map?.setFeatureState(
        { source: "geojson", id: selectedStateId.value },
        { selected: false }
      );
    }
    selectedStateId.value = e.features[0].id;

    mapHolder?.map?.setFeatureState(
      { source: "geojson", id: selectedStateId.value },
      { selected: true }
    );

    await loadBuildings(selectedStateId.value);
  }
}
let activeTouchId = null;

const createMouseEvent = (type, touch) =>
  new MouseEvent(type, {
    bubbles: true,
    cancelable: true,
    view: window,
    button: 0,
    buttons: type === "mouseup" ? 0 : 1,
    clientX: touch.clientX,
    clientY: touch.clientY,
    shiftKey: true
  });

const handleMouseDownCapture = e => {
  if (e.button !== 0 || e.shiftKey) return;

  Object.defineProperty(e, "shiftKey", {
    get: () => currentTool.value == "select"
  });
};

const mapEvents = {
  touchstart: "mousedown",
  touchmove: "mousemove",
  touchend: "mouseup"
};

const handleTouch = e => {
  if (currentTool.value != "select" || e.touches.length > 1) return;
  e.preventDefault();
  e.stopImmediatePropagation();

  const touch = e.changedTouches[0];
  e.target.dispatchEvent(
    new MouseEvent(mapEvents[e.type], {
      bubbles: true,
      clientX: touch.clientX,
      clientY: touch.clientY,
      shiftKey: true
    })
  );
};

// Do things here that require maplibre to be fully loaded.
watch(
  () => mapHolder.isLoaded,
  async isLoaded => {
    if (!isLoaded || !mapHolder.map) return;

    const rawMap = toRaw(mapHolder.map);
    rawMap.boxZoom.enable();

    const container = rawMap.getCanvasContainer();

    // Desktop mouse
    container.addEventListener("mousedown", handleMouseDownCapture, true);

    Object.keys(mapEvents).forEach(type =>
      container.addEventListener(type, handleTouch, {
        capture: true,
        passive: false
      })
    );

    // Keep track of retroed buildings so far.
    const res = await fetch(`api/retrofits/?user_id=${user_uuid.value}`, {
      method: "GET",
      // body: JSON.stringify({ coordinates: square, square_id: squareId }),
      headers: {
        "Content-Type": "application/json"
      }
    });

    retrofittedBuildings.value = JSON.parse(await res.json());

    // Mark the squares
    Object.values(retrofittedBuildings.value).forEach(building => {
      if (building.square_id != undefined) {
        mapHolder?.map?.setFeatureState(
          { source: "geojson", id: building.square_id },
          { retrofit: true }
        );
      }
    });

    console.log("RETRO BUILDINGS", retrofittedBuildings.value);
  },
  { immediate: true }
);
onMounted(async () => {
  if (user_uuid.value === "") {
    user_uuid.value = uuidv4();
  }

  const bdata = await (await fetch(boundaryUrl)).json();
  boundariesJSON.value = {
    type: "FeatureCollection",
    features: bdata.features.map(feature => {
      const point = center(feature);
      point.properties = { ...feature.properties };
      return point;
    })
  };

  socialJSON.value = await (await fetch(rotterdamUrl)).json();

  // const pyodide = await loadPyodide({
  //   indexURL: "https://cdn.jsdelivr.net/pyodide/v314.0.7/full/"
  // });
  // await pyodide.loadPackage("micropip");
  // const micropip = pyodide.pyimport("micropip");
  // await micropip.install("geopandas");
  // // const res = await fetch("kralingseveer_buildings.json");
  // const data = JSON.stringify(bdata);
  // pyodide.runPython(`
  //   import geopandas as gpd
  //   import json
  //   import io
  //   from shapely import Polygon, convex_hull
  //   gdf = gpd.read_file(io.StringIO('${data}'), fid_as_index=True).set_index("id")
  // `);
  // pyodideInstance.value = pyodide;
  // mapHolder?.map?.boxZoom.disable();
  if (mapHolder?.map) {
    // const handleMouseDown = e => {
    //   if (e.button !== 0 || !mapHolder?.map) return;

    //   // MapLibre's internal boxZoom checks `e.shiftKey` and `e.button === 0`
    //   const fakeShiftEvent = new MouseEvent("mousedown", {
    //     bubbles: true,
    //     cancelable: true,
    //     clientX: e.clientX,
    //     clientY: e.clientY,
    //     button: 0,
    //     buttons: 1,
    //     shiftKey: true
    //   });
    //   e.shiftKey = true;
    //   mapHolder?.map.boxZoom.mousedown(e, e.point);
    // };
    // const container = mapHolder.map.getCanvasContainer();
    // container.addEventListener("mousedown", handleMouseDown);

    //     const fakeShiftEvent = new MouseEvent('mousedown', {
    //   bubbles: true,
    //   cancelable: true,
    //   clientX: e.clientX,
    //   clientY: e.clientY,
    //   button: 0,
    //   buttons: 1,
    //   shiftKey: true,
    // });

    // rawMap.boxZoom._onMouseDown(fakeShiftEvent)

    mapHolder.map.boxZoom._boxZoomEnd = (mapInstance, p0, p1) => {
      const features = mapInstance.queryRenderedFeatures(
        [
          [Math.min(p0.x, p1.x), Math.min(p0.y, p1.y)],
          [Math.max(p0.x, p1.x), Math.max(p0.y, p1.y)]
        ],
        { layers: ["buildings"] }
      );
      const ids: number[] = [
        ...new Set(features.map(feature => feature.id).filter(id => id != null))
      ];

      selectBuildings(ids);
    };
  }
  // mapHolder?.map?.dragPan.disable();
});
</script>

<style lang="scss">
@import "maplibre-gl/dist/maplibre-gl.css";

.translucent {
  background-color: rgba(255, 255, 255, 0.05);
  box-shadow: 0 0 10px 5px rgba(250, 249, 249, 0.01) inset !important;
  pointer-events: none;
}
</style>
