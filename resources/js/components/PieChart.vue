<template>
  <loading-card :loading="loading" class="min-h-40 px-6 py-4">
    <div class="h-6 mb-4 flex items-center">
      <h4 class="mr-3 leading-tight text-sm font-bold">{{ checkTitle }}</h4>
      <div class="flex relative ml-auto flex-shrink-0">
        <default-button size="xs" class="mr-2" @click="fillData()" v-show="buttonRefresh">
          <icon-refresh />
        </default-button>
        <default-button size="xs" class="mr-2" @click="reloadPage()" v-show="buttonReload">
          <icon-refresh />
        </default-button>
        <default-button
          size="xs"
          class="mr-2"
          component="a"
          :href="externalLink"
          :target="externalLinkIn"
          v-show="btnExtLink"
        >
          <icon-external-link />
        </default-button>
        <select-control
          size="xxs"
          @change="handleFilterChanged"
          :selected="advanceFilterSelected"
          v-show="showAdvanceFilter"
        >
          <option v-for="filter in advanceFilter" v-bind:value="filter.value" :key="filter.key">
            {{ filter.text }}
          </option>
        </select-control>
      </div>
    </div>
    <line-chart
      v-if="!loading"
      :chart-data="datacollection"
      :options="options"
      :style="{ height: card.height && !['fixed', 'dynamic'].includes(card.height) ? card.height : 'auto' }"
    />
  </loading-card>
</template>

<script>
import LineChart from '../pie-chart.vue';
import IconRefresh from './Icons/IconRefresh';
import IconExternalLink from './Icons/IconExternalLink';

export default {
  components: {
    IconExternalLink,
    IconRefresh,
    LineChart,
  },
  data() {
    this.card.options = this.card.options != undefined ? this.card.options : false;

    // setup btn filter list
    const btnFilterList = this.card.options.btnFilterList;
    let filledAdvancedList = [];
    let i = 0;

    for (let index in btnFilterList) {
      filledAdvancedList[i] = { value: index, text: btnFilterList[index] };
      i++;
    }

    return {
      datacollection: {},
      options: {},
      loading: false,
      buttonRefresh: this.card.options.btnRefresh,
      buttonReload: this.card.options.btnReload,
      btnExtLink: this.card.options.extLink != undefined ? true : false,
      externalLink: this.card.options.extLink,
      externalLinkIn: this.card.options.extLinkIn != undefined ? this.card.options.extLinkIn : '_self',
      chartTooltips: this.card.options.tooltips != undefined ? this.card.options.tooltips : undefined,
      sweetAlert: this.card.options.sweetAlert2 != undefined ? this.card.options.sweetAlert2 : undefined,
      chartPlugins: this.card.options.plugins != undefined ? this.card.options.plugins : false,
      chartLayout: this.card.options.layout != undefined ? this.card.options.layout : {},
      chartLegend:
        this.card.options.legend != undefined
          ? this.card.options.legend
          : {
              display: true,
              position: 'right',
              labels: {
                fontColor: '#7c858e',
                fontFamily: "'Nunito'",
              },
            },
      showAdvanceFilter:
        this.card.model == 'custom' || this.card.model == undefined
          ? false
          : this.card.options.btnFilter == true
          ? true
          : false,
      advanceFilterSelected:
        this.card.options.btnFilterDefault != undefined ? this.card.options.btnFilterDefault : 'QTD',
      advanceFilter:
        this.card.options.btnFilterList != undefined
          ? filledAdvancedList
          : [
              { text: 'Year to Date', value: 'YTD' },
              { text: 'Quarter to Date', value: 'QTD' },
              { text: 'Month to Date', value: 'MTD' },
              { text: '30 Days', value: 30 },
              { text: '60 Days', value: 60 },
              { text: '365 Days', value: 365 },
            ],
    };
  },
  computed: {
    checkTitle() {
      return this.card.title !== undefined ? this.card.title : 'Chart JS Integration';
    },
  },
  props: ['card'],
  mounted() {
    this.fillData();
  },
  methods: {
    reloadPage() {
      window.location.reload();
    },

    handleFilterChanged(value) {
      this.advanceFilterSelected = value;
      this.fillData();
    },
    fillData() {
      this.options = {
        ...this.card.options,
        layout: this.chartLayout,
        responsive: true,
        maintainAspectRatio: false,
        plugins: {
          legend: this.chartLegend,
          ...this.chartPlugins,
        },
      };

      if (this.chartTooltips !== undefined) {
        this.options.plugins.tooltip = this.chartTooltips;
        const tooltiplist = ['custom', 'itemSort', 'filter'];
        for (let z = 0; z < tooltiplist.length; z++) {
          if (this.options.plugins.tooltip[tooltiplist[z]] != undefined) {
            if (this.options.plugins.tooltip[tooltiplist[z]].search('function') != -1) {
              eval(
                'this.options.plugins.tooltip.' + tooltiplist[z] + ' = ' + this.options.plugins.tooltip[tooltiplist[z]]
              );
            }
          }
        }

        if (this.chartTooltips.callbacks !== undefined) {
          const callbacklist = [
            'beforeTitle',
            'title',
            'afterTitle',
            'beforeBody',
            'beforeLabel',
            'label',
            'labelColor',
            'labelTextColor',
            'afterLabel',
            'afterBody',
            'beforeFooter',
            'footer',
            'afterFooter',
          ];
          for (let i = 0; i < callbacklist.length; i++) {
            if (this.options.plugins?.tooltip?.callbacks?.[callbacklist[i]] != undefined) {
              if (this.options.plugins.tooltip.callbacks[callbacklist[i]].search('function') != -1) {
                eval(
                  'this.options.plugins.tooltip.callbacks.' +
                    callbacklist[i] +
                    ' = ' +
                    this.options.plugins.tooltip.callbacks[callbacklist[i]]
                );
              }
            }
          }
        }
      }

      if (this.card.model == 'custom' || this.card.model == undefined) {
        // Custom Data
        this.title = this.card.title;
        this.datacollection = {
          labels: this.card.options.xaxis.categories,
          datasets: this.card.series,
        };

        // START == SETUP POPUP
        const sweetAlertWithLink = this.sweetAlert;
        if (sweetAlertWithLink != undefined) {
          this.options.onClick = function (event, element) {
            if (element.length > 0) {
              let series = element[0].datasetLabel;
              let label = element[0].label;
              let value = this.data.datasets[element[0].datasetIndex].data[element[0].index];

              const toLink =
                sweetAlertWithLink.linkTo != undefined
                  ? sweetAlertWithLink.linkTo
                  : 'https://coroo.github.io/nova-chartjs/';
              const { linkTo, ...sweetAlert } = sweetAlertWithLink;

              // sum data
              let dataArr = this.data.datasets[0].data;
              let sum = 0;
              sum = dataArr.reduce((a, b) => parseInt(a) + parseInt(b), 0);
              let percentage = (value / sum) * 100;

              const Swal = require('sweetalert2');
              Swal.fire({
                title: sweetAlert.title != undefined ? sweetAlert.title : '<strong>' + label + '</strong>',
                icon: sweetAlert.icon != undefined ? sweetAlert.icon : 'info',
                html:
                  sweetAlert.html != undefined
                    ? sweetAlert.html
                    : 'Percentage: <b>' +
                      percentage.toFixed(2) +
                      '%</b><br/><b>' +
                      value +
                      '</b> data from <b>' +
                      sum +
                      '</b><br/> ',
                showCloseButton: sweetAlert.showCloseButton != undefined ? sweetAlert.showCloseButton : true,
                showCancelButton: sweetAlert.showCancelButton != undefined ? sweetAlert.showCancelButton : true,
                focusConfirm: sweetAlert.focusConfirm != undefined ? sweetAlert.focusConfirm : false,
                confirmButtonText:
                  sweetAlert.confirmButtonText != undefined
                    ? sweetAlert.confirmButtonText
                    : '<i class="fas fa-external-link-alt"></i> See Detail',
                confirmButtonAriaLabel:
                  sweetAlert.confirmButtonAriaLabel != undefined ? sweetAlert.confirmButtonAriaLabel : 'See Detail',
                cancelButtonAriaLabel:
                  sweetAlert.cancelButtonAriaLabel != undefined ? sweetAlert.cancelButtonAriaLabel : 'Cancel',
                footer:
                  sweetAlert.footer != undefined
                    ? sweetAlert.footer
                    : '<a href="https://coroo.github.io/nova-chartjs/" target="_blank" style="text-decoration:none; color:#777; font-size:14px">Nova Chart JS © ' +
                      new Date().getFullYear() +
                      '</a>',
                ...sweetAlert,
              }).then((result) => {
                if (result.value) {
                  window.location = toLink;
                }
              });
            }
          };
        }
        // END == SETUP POPUP

        if (this.card.options.showPercentage != undefined) {
          if (this.card.options.showPercentage == true) {
            let dataArr = this.card.series[0].data;
            let sum = dataArr.reduce((a, b) => parseInt(a) + parseInt(b), 0);
            this.options.plugins.tooltip = {
              callbacks: {
                ...(this.options.plugins?.tooltip?.callbacks || {}),
                label: function (context) {
                  return context.label + ': ' + '' + context.raw + ' (' + ((context.raw * 100) / sum).toFixed(2) + '%)';
                },
              },
            };
          }
        }
      } else {
        if (this.showAdvanceFilter == true)
          this.card.options.advanceFilterSelected =
            this.advanceFilterSelected != undefined ? this.advanceFilterSelected : false;

        // Use Model
        this.loading = true;
        Nova.request()
          .get('/nova-vendor/coroowicaksono/check-data/circle-endpoint', {
            params: {
              model: this.card.model,
              series: this.card.series,
              options: this.card.options,
              join: this.card.join,
              col_xaxis: this.card.col_xaxis,
              expires: 0,
            },
          })
          .then(({ data }) => {
            this.datacollection = {
              labels: data.dataset.xAxis,
              datasets: data.dataset.yAxis,
            };

            this.loading = false;

            // START == SETUP POPUP
            const sweetAlertWithLink = this.sweetAlert;
            if (sweetAlertWithLink != undefined) {
              this.options.onClick = function (event, element) {
                if (element.length > 0) {
                  let series = element[0].datasetLabel;
                  let label = element[0].label;
                  let value = this.data.datasets[element[0].datasetIndex].data[element[0].index];

                  const toLink =
                    sweetAlertWithLink.linkTo != undefined
                      ? sweetAlertWithLink.linkTo
                      : 'https://coroo.github.io/nova-chartjs/';
                  const { linkTo, ...sweetAlert } = sweetAlertWithLink;

                  // sum data
                  let dataArr = this.data.datasets[0].data;
                  let sum = dataArr.reduce((a, b) => parseInt(a) + parseInt(b), 0);
                  let percentage = (value / sum) * 100;

                  const Swal = require('sweetalert2');
                  Swal.fire({
                    title: sweetAlert.title != undefined ? sweetAlert.title : '<strong>' + label + '</strong>',
                    icon: sweetAlert.icon != undefined ? sweetAlert.icon : 'info',
                    html:
                      sweetAlert.html != undefined
                        ? sweetAlert.html
                        : 'Percentage: <b>' +
                          percentage.toFixed(2) +
                          '%</b><br/><b>' +
                          value +
                          '</b> data from <b>' +
                          sum +
                          '</b><br/> ',
                    showCloseButton: sweetAlert.showCloseButton != undefined ? sweetAlert.showCloseButton : true,
                    showCancelButton: sweetAlert.showCancelButton != undefined ? sweetAlert.showCancelButton : true,
                    focusConfirm: sweetAlert.focusConfirm != undefined ? sweetAlert.focusConfirm : false,
                    confirmButtonText:
                      sweetAlert.confirmButtonText != undefined
                        ? sweetAlert.confirmButtonText
                        : '<i class="fas fa-external-link-alt"></i> See Detail',
                    confirmButtonAriaLabel:
                      sweetAlert.confirmButtonAriaLabel != undefined ? sweetAlert.confirmButtonAriaLabel : 'See Detail',
                    cancelButtonAriaLabel:
                      sweetAlert.cancelButtonAriaLabel != undefined ? sweetAlert.cancelButtonAriaLabel : 'Cancel',
                    footer:
                      sweetAlert.footer != undefined
                        ? sweetAlert.footer
                        : '<a href="https://coroo.github.io/nova-chartjs/" target="_blank" style="text-decoration:none; color:#777; font-size:14px">Nova Chart JS © ' +
                          new Date().getFullYear() +
                          '</a>',
                    ...sweetAlert,
                  }).then((result) => {
                    if (result.value) {
                      window.location = toLink;
                    }
                  });
                }
              };
            }
            // END == SETUP POPUP

            if (this.card.options.showPercentage != undefined) {
              if (this.card.options.showPercentage == true) {
                let dataArr = data.dataset.yAxis[0].data;
                let sum = dataArr.reduce((a, b) => parseInt(a) + parseInt(b), 0);
                this.options.plugins.tooltip = {
                  callbacks: {
                    ...(this.options.plugins?.tooltip?.callbacks || {}),
                    label: function (context) {
                      return (
                        context.label + ': ' + '' + context.raw + ' (' + ((context.raw * 100) / sum).toFixed(2) + '%)'
                      );
                    },
                  },
                };
              }
            }
          })
          .catch(({ response }) => {
            this.errors = response.data.errors;
            this.loading = false;
          });
      }
    },
  },
};
</script>
