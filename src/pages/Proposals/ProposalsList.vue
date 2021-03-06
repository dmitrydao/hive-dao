<template>
  <div>
    <!-- PROPOSALS MODALS -->
    <b-modal
      ref="modal-voting"
      :title="`${$t('vote.supportingProposal')}`"
      centered
      hide-footer
    >
      <VotingModal
        :proposalIdProp="proposalId"
        :userProp="user.name"
        :voteStatusProp="user.loggedIn ? isApproved(proposalId) : voteStatus"
        :loggedInProp="user.loggedIn"
        :hivesigner="true"
        :shareonsocial="true"
      />
    </b-modal>

    <b-modal
      size="lg"
      scrollable
      ref="modal-voters2"
      :title="`${$t('proposals.proposalVoters')} (#${proposalId})`"
      centered
      hide-footer
    >
      <SkeletonLoading v-if="!accounts.length" />
      <VotersModal :accounts="accounts" :proposalVoters="proposalVoters" />
    </b-modal>

    <b-modal
      size="lg"
      scrollable
      ref="modal-post"
      :title="`${proposalSubject} (#${proposalId})`"
      centered
      hide-footer
    >
      <SkeletonLoading v-if="!post.body" />
      <div v-if="post.body">
        <vue-markdown :source="post.body" class="postImage"></vue-markdown>
        <h3>{{ $t("common.originalPostTitle") }}</h3>
        <div>
          {{ $t("common.originalPostDescription") }}
          <a
            :href="`https://peakd.com/@${post.author}/${post.permlink}`"
            target="_blank"
            >PeakD</a
          >.
        </div>
      </div>
    </b-modal>
    <!-- PROPOSALS TABLE -->
    <div class="row mt-2">
      <!-- <div class="row mt-2 d-none d-md-block"> -->
      <div class="col-md-12">
        <b-table
          :class="
            `table table-padded table-${votesStatusProp} align-items-center`
          "
          :show-empty="true"
          :items="proposals(votesStatusProp, `${statusProp}`)"
          :fields="fieldsProposals"
          :filter="filterProp"
          responsive="sm"
          :sort-by.sync="proposalsSortBy"
          :sort-desc.sync="proposalsSortDesc"
          :sort-direction="proposalsSortDirection"
        >
          <template v-slot:empty>
            <div class="d-flex justify-content-center">
              <span class="text-center mt-1">{{
                $t("proposals.emptyState")
              }}</span>
            </div>
          </template>

          <!-- Votes -->
          <template v-slot:cell(total_votes)="data">
            <span style="cursor:pointer" @click="loadVoters(data.item.id)"
              >{{ data.item.total_votes | numeric }} HP</span
            >
          </template>
          <!-- Status -->
          <template v-slot:cell(status)="data">
            <span class="badge badge-dot">
              <i :class="`bg-${votesStatusProp}`"></i>
            </span>
            <span v-if="data.item.status === 'expired'"
              >{{ $t("common.completedProposalsLabel") }}:<br />
              ({{ data.item.end_date | dateFilter }})</span
            >
            <span v-if="data.item.status !== 'expired'">{{
              data.item.status === "active"
                ? $t("common.activeProposalsLabel")
                : $t("common.inactiveProposalsLabel")
            }}</span
            ><br />
            <span v-if="data.item.status !== 'expired'"
              >({{ $t("common.ends") }}
              {{ data.item.end_date | daysLeft }})</span
            >
          </template>
          <!-- Description -->
          <template v-slot:cell(description)="data">
            <div class="row">
              <div
                class="media align-items-center col-md-2 d-none d-md-block mr-3"
              >
                <div class="avatar rounded-circle">
                  <router-link :to="'/proposals/' + data.item.creator">
                    <img
                      :src="
                        `https://images.hive.blog/u/${data.item.creator}/avatar`
                      "
                    />
                  </router-link>
                </div>
              </div>
              <div class="col-md-9">
                <div class="text-wrap">
                  <span
                    class="text-dark h6"
                    style="cursor:pointer"
                    @click="
                      showPostModal(
                        data.item.id,
                        data.item.creator,
                        data.item.permlink_short,
                        data.item.subject
                      )
                    "
                    >{{ data.item.subject }}</span
                  >
                  <span
                    class="badge badge-success"
                    v-if="data.item.refunding === true"
                    style="cursor:pointer"
                    v-b-tooltip.hover
                    :title="`${$t('proposals.returnProposalInfo1')}`"
                  >
                    {{ $t("proposals.returnProposal") }}</span
                  >
                  <span
                    class="badge badge-warning"
                    v-if="data.item.burning === true"
                    style="cursor:pointer"
                    v-b-tooltip.hover
                    :title="`${$t('proposals.burnProposalInfo')}`"
                    >{{ $t("proposals.burnProposal") }}</span
                  >
                </div>
                <div>
                  {{ $t("common.by") }}
                  <router-link
                    class="text-uppercase text-muted"
                    :to="'/proposals/' + data.item.creator"
                    >@{{ data.item.creator }}</router-link
                  >
                  <span v-if="data.item.creator !== data.item.receiver">
                    {{ $t("common.for") }}
                    <router-link
                      class="text-uppercase text-muted"
                      :to="'/proposals/' + data.item.receiver"
                      >@{{ data.item.receiver }}</router-link
                    >
                  </span>
                </div>
              </div>
            </div>
          </template>
          <!-- Duration -->
          <template v-slot:cell(duration)="data">
            {{ data.item.duration | numeric3 }} {{ $t("common.days") }}
          </template>
          <!-- Requested -->
          <template v-slot:cell(requested)="data">
            <div>{{ data.item.total_requested | numeric }} HBD</div>
            <div>
              {{ data.item.daily_pay | numeric }} HBD /
              {{ $t("common.day") }}
            </div>
          </template>
          <template v-slot:cell(funding)="data">
            <div
              v-b-tooltip.hover
              :title="
                `${Number(
                  data.item.funding.availableBudget
                ).toLocaleString()} HBD ${$t('common.availableBudget')}`
              "
            >
              {{ data.item.funding.fundedStake }}%
            </div>
          </template>
          <!-- Voting -->
          <template v-slot:cell(vote)="data">
            <button
              v-if="!user.loggedIn"
              class="btn btn-sm btn-light text-dark"
              @click="showVotingModal(data.item.id)"
            >
              <i class="far fa-arrow-alt-circle-up"></i>
            </button>
            <div
              style="cursor:pointer"
              v-if="user.loggedIn"
              @click="showVotingModal(data.item.id)"
              :class="
                `d-inline-block link link-underline-${
                  isApproved(data.item.id) ? 'inactive' : 'active'
                } text-${isApproved(data.item.id) ? 'warning' : 'success'} mt-4`
              "
            >
              <i
                :class="
                  `far fa-arrow-alt-circle-${
                    isApproved(data.item.id) ? 'down' : 'up'
                  } mr-1`
                "
              ></i>
              {{
                isApproved(data.item.id)
                  ? $t("keychain.voteUnsupport")
                  : $t("keychain.voteSupport")
              }}
            </div>
          </template>
        </b-table>
      </div>
    </div>
  </div>
</template>

<script>
import { mapState, mapGetters } from "vuex";
import { i18n } from "@/utils/plugins/i18n.js";
import VueMarkdown from "vue-markdown";
import { DefaultRenderer } from "steem-content-renderer";
import VotingModal from "@/components/VotingModal";
import SkeletonLoading from "@/components/SkeletonLoading";
import VotersModal from "@/components/VotersModal";

export default {
  components: {
    VueMarkdown,
    DefaultRenderer,
    VotingModal,
    SkeletonLoading,
    VotersModal
  },
  computed: {
    ...mapState([
      "voters",
      "accounts",
      "dailyBudget",
      "globalProperties",
      "language",
      "proposalVoters",
      "post",
      "user",
      "voterProposals"
    ]),
    ...mapGetters({
      proposals: "proposalsByVotesStatus",
      totalProposalsByVotesStatus: "totalProposalsByVotesStatus",
      totalProposals: "totalProposals"
    })
  },
  props: {
    filterProp: String,
    votesStatusProp: String,
    statusProp: String
  },
  methods: {
    isApproved(id) {
      if (this.voterProposals.includes(id)) {
        return true;
      } else {
        return false;
      }
    },
    loadVoters(id) {
      this.proposalId = id;
      this.$refs["modal-voters2"].show();
      this.$store.dispatch("fetchProposalVoters", id);
    },
    showVotingModal(id) {
      this.proposalId = id;
      this.$refs["modal-voting"].show();
    },
    showPostModal(id, creator, permlink, subject) {
      this.proposalSubject = subject;
      this.proposalId = id;
      const renderer = new DefaultRenderer({
        baseUrl: "https://peakd.com/",
        breaks: true,
        skipSanitization: false,
        allowInsecureScriptTags: false,
        addNofollowToLinks: true,
        doNotShowImages: false,
        ipfsPrefix: "",
        assetsWidth: 640,
        assetsHeight: 480,
        imageProxyFn: url => url,
        usertagUrlFn: account => "/@" + account,
        hashtagUrlFn: hashtag => "/trending/" + hashtag,
        isLinkSafeFn: url => true
      });
      this.$refs["modal-post"].show();
      this.$store.dispatch("fetchPost", [creator, permlink]).then(() => {
        this.post.body = renderer.render(this.post.body);
      });
    }
  },
  data() {
    return {
      fieldsProposals: [
        {
          key: "total_votes",
          label: i18n.t("common.totalVotes")
        },
        {
          key: "description",
          label: i18n.t("common.description")
        },
        {
          key: "status",
          label: i18n.t("common.status")
        },
        {
          key: "duration",
          label: i18n.t("common.duration")
        },
        {
          key: "requested",
          label: i18n.t("common.requested")
        },
        {
          key: "funding",
          label: i18n.t("common.funding")
        },
        {
          key: "vote",
          label: i18n.t("common.vote")
        }
      ],
      proposalsSortBy: "total_votes",
      proposalsSortDesc: true,
      proposalsSortDirection: "asc",
      voteStatus: false,
      proposalId: 0,
      proposalSubject: ""
    };
  }
};
</script>

<style></style>
