<template>
  <Row type="flex" :gutter="18">
    <Col :span=19>
      <Panel shadow>
        <div slot="title">{{$t('m.Problem_List')}}</div>
        <div slot="extra">
          <ul class="filter">
            <li>
              <Dropdown @on-click="filterByDifficulty" style="cursor:pointer;">
                <span>{{query.difficulty === '' ? this.$i18n.t('m.Difficulty') : this.$i18n.t('m.' + query.difficulty)}}
                  <Icon type="md-arrow-dropdown"></Icon>
                </span>
                <Dropdown-menu slot="list">
                  <Dropdown-item name="">{{$t('m.All')}}</Dropdown-item>
                  <Dropdown-item name="Low">{{$t('m.Low')}}</Dropdown-item>
                  <Dropdown-item name="Mid" >{{$t('m.Mid')}}</Dropdown-item>
                  <Dropdown-item name="High">{{$t('m.High')}}</Dropdown-item>
                </Dropdown-menu>
              </Dropdown>
            </li>
            <li>
              <i-switch size="large" @on-change="handleTagsVisible">
                <span slot="open">{{$t('m.Tags')}}</span>
                <span slot="close">{{$t('m.Tags')}}</span>
              </i-switch>
            </li>
            <li>
              <Input v-model="query.keyword"
                    @on-enter="filterByKeyword"
                    @on-click="filterByKeyword"
                    placeholder="Từ khóa"
                    icon="ios-search"/>
            </li>
            <li>
              <Button type="info" @click="onReset">
                <Icon type="md-refresh"></Icon>
                {{$t('m.Reset')}}
              </Button>
            </li>
          </ul>
        </div>
        <Table style="width: 100%; font-size: 16px;"
              :columns="problemTableColumns"
              @on-sort-change="handleSortChange"
              :data="problemList"
              :loading="loadings.table"
              disabled-hover></Table>
      </Panel>
      <Pagination :total="total" :page-size="limit" @on-change="pushRouter" :current.sync="query.page"></Pagination>
    </Col>

    <Col :span="5">
      <Panel :padding="10">
        <div slot="title" class="taglist-title">{{$t('m.TagsTitle')}}</div>
        <Button long id="pick-one" @click="pickone">
          <Icon type="shuffle"></Icon>
          {{$t('m.Pick_One')}}
        </Button>
        <br>

        <Button v-for="tag in tagList"
                :key="tag.name"
                @click="filterByTag(tag.name)"
                :disabled="query.tag === tag.name"
                shape="circle"
                class="tag-btn">{{tag.name}}
        </Button>
      </Panel>
      <Spin v-if="loadings.tag" fix size="large"></Spin>
    </Col>
  </Row>
</template>

<script>
  import { mapGetters } from 'vuex'
  import api from '@oj/api'
  import utils from '@/utils/utils'
  import { ProblemMixin } from '@oj/components/mixins'
  import Pagination from '@oj/components/Pagination'

  export default {
    name: 'ProblemList',
    mixins: [ProblemMixin],
    components: {
      Pagination
    },
    data () {
      return {
        tagList: [],
        problemTableColumns: [
          {
            title: 'ID',
            key: '_id',
            width: 120,
            render: (h, params) => {
              return h('a', {
                props: {
                  type: 'text',
                  size: 'large'
                },
                attrs: {
                  href: '/problem/' + params.row._id
                },
                style: {
                  'padding': '2px 0px',
                  fontSize: '14px',
                  color: '#495060'
                }
              }, params.row._id)
            }
          },
          {
            title: this.$i18n.t('m.Title'),
            width: 400,
            render: (h, params) => {
              return h('a', {
                props: {
                  type: 'text',
                  size: 'large'
                },
                attrs: {
                  href: '/problem/' + params.row._id
                },
                style: {
                  padding: '2px 0',
                  overflow: 'hidden',
                  whiteSpace: 'nowrap',
                  textOverflow: 'ellipsis',
                  textAlign: 'left',
                  width: '95%',
                  fontSize: '14px',
                  color: '#495060'
                }
              }, params.row.title)
            }
          },
          {
            title: this.$i18n.t('m.Level'),
            render: (h, params) => {
              let t = params.row.difficulty
              let color = 'primary'
              if (t === 'Low') color = 'success'
              else if (t === 'High') color = 'warning'
              return h('Tag', {
                props: {
                  color: color
                }
              }, this.$i18n.t('m.' + params.row.difficulty))
            }
          },
          {
            sortable: 'custom',
            title: this.$i18n.t('m.Total'),
            key: 'submission_number'
          },
          {
            sortable: 'custom',
            title: this.$i18n.t('m.AC_Count'),
            key: 'accepted_number',
            render: (h, params) => {
              return h('span', params.row.accepted_number)
            }
          }
        ],
        problemList: [],
        limit: 20,
        total: 0,
        loadings: {
          table: true,
          tag: true
        },
        routeName: '',
        query: {
          keyword: '',
          difficulty: '',
          tag: '',
          page: 1,
          orderby: null
        }
      }
    },
    mounted () {
      this.init()
    },
    methods: {
      init (simulate = false) {
        this.routeName = this.$route.name
        let query = this.$route.query
        this.query.difficulty = query.difficulty || ''
        this.query.keyword = query.keyword || ''
        this.query.tag = query.tag || ''
        this.query.page = parseInt(query.page) || 1
        if (this.query.page < 1) {
          this.query.page = 1
        }
        this.query.limit = parseInt(query.limit) || this.limit
        if (!simulate) {
          this.getTagList()
        }
        this.getProblemList()
      },
      pushRouter () {
        this.$router.push({
          name: 'problem-list',
          query: utils.filterEmptyValue(this.query)
        })
      },
      getProblemList () {
        let offset = (this.query.page - 1) * this.query.limit
        this.loadings.table = true
        api.getProblemList(offset, this.limit, this.query).then(res => {
          this.loadings.table = false
          this.total = res.data.data.total
          this.problemList = res.data.data.results
          for (let i = 0; i < this.problemList.length; i++) {
            this.problemList[i].ac_rate = this.problemList[i].submission_number === 0 ? 0.00 : (this.problemList[i].accepted_number / this.problemList[i].submission_number * 100)
          }
          if (this.isAuthenticated) {
            this.addStatusColumn(this.problemTableColumns, res.data.data.results)
          }
        }, res => {
          this.loadings.table = false
        })
      },
      getTagList () {
        api.getProblemTagList().then(res => {
          this.tagList = res.data.data.sort((a, b) => {
            // return a.id - b.id
            return a.name.localeCompare(b.name)
          })
          this.loadings.tag = false
        }, res => {
          this.loadings.tag = false
        })
      },
      handleSortChange (data) {
        let key = data['key']
        if (data['order'] === 'desc') {
          key = '-' + key
        }
        if (data['order'] !== 'normal') {
          this.query.orderby = key
        } else {
          this.query.orderby = null
        }
        this.pushRouter()
      },
      filterByTag (tagName) {
        this.query.tag = tagName
        this.query.page = null
        this.pushRouter()
      },
      filterByDifficulty (difficulty) {
        this.query.difficulty = difficulty
        this.query.page = null
        this.pushRouter()
      },
      filterByKeyword () {
        this.query.page = null
        this.pushRouter()
      },
      handleTagsVisible (value) {
        if (value) {
          this.problemTableColumns.push(
            {
              title: this.$i18n.t('m.Tags'),
              align: 'center',
              render: (h, params) => {
                let tags = []
                params.row.tags.forEach(tag => {
                  tags.push(h('Tag', {}, tag))
                })
                return h('div', {
                  style: {
                    margin: '8px 0'
                  }
                }, tags)
              }
            })
        } else {
          this.problemTableColumns.splice(this.problemTableColumns.length - 1, 1)
        }
      },
      onReset () {
        this.$router.push({name: 'problem-list'})
      },
      pickone () {
        api.pickone().then(res => {
          this.$success('Chúc AC nhé ^^')
          this.$router.push({name: 'problem-details', params: {problemID: res.data.data}})
        })
      }
    },
    computed: {
      ...mapGetters(['isAuthenticated'])
    },
    watch: {
      '$route' (newVal, oldVal) {
        if (newVal !== oldVal) {
          this.init(true)
        }
      },
      'isAuthenticated' (newVal) {
        if (newVal === true) {
          this.init()
        }
      }
    }
  }
</script>

<style scoped lang="less">
  .taglist-title {
    margin-left: -10px;
    margin-bottom: -10px;
  }

  .tag-btn {
    margin-right: 5px;
    margin-bottom: 10px;
  }

  #pick-one {
    margin-bottom: 10px;
  }
</style>
