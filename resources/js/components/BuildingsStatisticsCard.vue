<template>
    <el-row style="margin-bottom: 4px;" :gutter="20" v-if="data.card_data">
        <el-col :sm="24" :md="12" :lg="6">
            <el-card class="box-card-count" style="margin-bottom: 17px;">
                <div class="total-wrapper">
                    <div class="total-box-card-header">
                        <span>{{ $t('dashboard.buildings.total_building') }}</span>
                    </div>
                    <div class="total-box-card-body">
                        <div class="box-card-count">
                            {{ data.total_buildings }}
                        </div>
                    </div>
                </div>
            </el-card>
        </el-col>
        <el-col :sm="24" :md="12" :lg="6">
            <el-card class="box-card" :style="{'border-color': getUnitsCountColor('total_units', 'name')}" style="margin-bottom: 20px;">
                <div slot="header" class="box-card-header clearfix">
                    <span>{{ $t('dashboard.buildings.total_units') }}</span>
                </div>
                <div class="box-card-body">
                    <div class="box-card-count">
                        {{ data.card_data.total_units }}
                    </div>
                    <div class="box-card-progress">
                        <el-progress type="circle" :percentage="100" :width="70" :color="getUnitsCountColor('total_units', 'name')" :stroke-width="5"></el-progress>
                    </div>
                </div>
            </el-card>
        </el-col>
        <el-col :sm="24" :md="12" :lg="6" v-for="n in 2" :key="n">
            <el-card class="box-card" :style="{'border-color': getUnitsCountColor(data.card_data.labels[n - 1], 'name')}" style="margin-bottom: 20px;">
                <div slot="header" class="box-card-header clearfix">
                    <span>{{ $t('dashboard.buildings.' + data.card_data.labels[n - 1]) }}</span>
                </div>
                <div class="box-card-body">
                    <div class="box-card-count">
                        {{ data.card_data.data[n - 1] }}
                    </div>
                    <div class="box-card-progress">
                        <el-progress type="circle" :percentage="percentage[n - 1]" :width="70" :color="getUnitsCountColor(data.card_data.labels[n - 1], 'name')" :stroke-width="5"></el-progress>
                    </div>
                </div>
            </el-card>
        </el-col>
    </el-row>
</template>

<script>
    import Card from './Card';
    import globalFunction from "helpers/globalFunction";
    export default {
        name: 'BuildingsStatisticsCard',
        mixins: [globalFunction],
        components: {
            Card
        },
        props: {
            data: {
                type: Object,
                required: true
            },
            animationTrigger: {
                type: String
            }
        },
        data() {
            return {
                percentage: []
            }
        },
        watch: {
            animationTrigger: function(val) {
                const that = this;
                if (val == 'buildings' && this.data.card_data.tag_percentage) {
                    this.percentage = this.data.card_data.tag_percentage.map(val => 0);
                    setTimeout(function() {
                        that.percentage = that.data.card_data.tag_percentage;
                    }, 200);
                }
            }
        }
    }
</script>

<style lang="scss" scoped>
    .total-wrapper {
        padding: 5px 0 0px 0;
    }
    .avg-wrapper {
        padding: 11px 0 5px 0;
    }
    .el-divider.el-divider--horizontal {
        width: 90%;
        margin: 0 auto;
    }
    .el-card {
        :global(.el-card__body) {
            padding: 0;
            .el-row {
                margin: -1px;
                margin-bottom: -2px;
                margin-right: -2px;
                border: 1px darken(#fff, 4%) solid;
                .el-col {
                    position: relative;
                    border-right: 1px darken(#fff, 4%) solid;
                    border-bottom: 1px darken(#fff, 4%) solid;
                    padding: 1em;
                    text-align: center;
                    display: flex;
                    align-items: center;
                    justify-content: center;
                    i {
                        border-radius: 50%;
                        line-height: 40px;
                        margin-bottom: 1em;
                        text-align: center;
                        background-color: darken(#fff, 4%);
                        width: 40px;
                        height: 40px;
                    }
                    h3 {
                        margin: 0;
                        font-size: 1.8em;
                        small {
                            font-size: 40%;
                            font-weight: normal;
                            display: block;
                            text-transform: uppercase;
                            color: darken(#fff, 24%);
                        }
                    }
                }
            }
        }
    }
</style>
