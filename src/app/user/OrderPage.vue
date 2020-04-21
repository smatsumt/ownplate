<template>
  <div>
    <template v-if="notFound">
      <not-found />
    </template>
    <template v-else>
      <section class="section">
        <div v-if="paid">
          <div style="text-align: center;">
            <i class="far fa-check-circle thankyou-icon"></i>
            <p class="thankyou">
              {{$t('order.thankyou')}}
              <br />
              {{$t('order.pleaseStay')}}
            </p>
          </div>
          <h2>{{ $t('order.orderStatus') + orderName }}</h2>
          <div v-if="paid" style="text-align: center;">
            <p
              :class="orderStatusKey"
              style="margin-bottom:1rem;padding:0.5rem"
            >{{ $t("order.status." + orderStatusKey) }}</p>
            <b-button
              v-if="just_paid"
              type="is-danger"
              :loading="isCanceling"
              @click="handleCancelPayment"
              style="margin-bottom:1rem"
            >{{$t('button.cancel')}}</b-button>
          </div>
        </div>
        <shop-orner-info
          :src="this.shopInfo.restProfilePhoto"
          :name="this.shopInfo.restaurantName"
        />
        <shop-info v-if="paid" :compact="true" :shopInfo="shopInfo" />

        <h2>{{ $t('order.yourOrder') + ": " + orderName }}</h2>
        <order-info
          :orderItems="this.orderItems"
          :orderInfo="this.orderInfo||{}"
          @change="handleTipChange"
        ></order-info>

        <b-notification :closable="false" v-if="newOrder">
          {{$t('order.validating')}}
          <b-loading :is-full-page="false" :active.sync="newOrder" :can-cancel="true"></b-loading>
        </b-notification>

        <div v-if="just_validated">
          <div class="is-centered" style="text-align: center;">
            <b-button
              expanded
              rounded
              :loading="isDeleting"
              @click="handleEditItems"
              style="margin-bottom:1rem;"
            >{{$t('order.editItems')}}</b-button>
          </div>

          <hr class="hr-black" />
          <div v-if="!hidePayment">
            <h2>{{$t('order.yourPayment')}}</h2>
            <stripe-card
              :stripe-account="this.stripeAccount"
              @change="handleCardStateChange"
              ref="stripe"
            ></stripe-card>
            <!-- <credit-card-input></credit-card-input> -->

            <div class="is-centered" style="text-align: center;">
              <b-button
                type="is-primary"
                expanded
                rounded
                :loading="isPaying"
                :disabled="!cardState.complete"
                style="margin-top:4rem;padding-top: 0.2rem;"
                size="is-large"
                @click="handlePayment"
              >
                <span
                  class="p-font bold"
                >{{$t('order.placeOrder')}} {{$n(orderInfo.total + tip, 'currency')}}</span>
              </b-button>
            </div>
          </div>
          <div class="is-centered" style="text-align: center;">
            <b-button
              expanded
              :type="hidePayment ? 'is-primary' : ''"
              rounded
              style="margin-top:4rem;padding-top: 0.2rem;"
              size="is-large"
              @click="handleNoPayment"
            >
              <span class="p-font bold">{{$t('order.placeOrderNoPayment')}}</span>
            </b-button>
          </div>
        </div>
      </section>
    </template>
  </div>
</template>

<script>
import ShopOrnerInfo from "~/app/user/Restaurant/ShopOrnerInfo";
import OrderInfo from "~/app/user/Order/OrderInfo";
import ShopInfo from "~/app/user/Restaurant/ShopInfo";
import StripeCard from "~/app/user/Order/StripeCard";
import NotFound from "~/components/NotFound";

import { db, firestore, functions } from "~/plugins/firebase.js";
import { order_status } from "~/plugins/constant.js";
import { nameOfOrder } from "~/plugins/strings.js";
import { releaseConfig } from "~/plugins/config.js";
import { checkoutCreate, checkoutCancel } from "~/plugins/stripe.js";

export default {
  name: "Order",
  components: {
    ShopOrnerInfo,
    OrderInfo,
    ShopInfo,
    StripeCard,
    NotFound
  },
  data() {
    return {
      isPaying: false,
      restaurantsId: this.restaurantId(),
      shopInfo: { restaurantName: "" },
      cardState: {},
      stripeAccount: null,
      orderInfo: {},
      menus: [],
      detacher: [],
      isDeleting: false,
      tip: 0,
      isCanceling: false,
      notFound: false
    };
  },
  created() {
    const restaurant_detacher = db
      .doc(`restaurants/${this.restaurantId()}`)
      .onSnapshot(async restaurant => {
        if (restaurant.exists) {
          const restaurant_data = restaurant.data();
          this.shopInfo = restaurant_data;
          const uid = restaurant_data.uid;
          const snapshot = await db.doc(`/admins/${uid}/public/stripe`).get();
          this.stripeAccount = snapshot.data()["stripeAccount"];
        } else {
          this.notFound = true;
        }
      });
    const menu_detacher = db
      .collection(`restaurants/${this.restaurantId()}/menus`)
      .onSnapshot(menu => {
        if (!menu.empty) {
          this.menus = menu.docs.map(this.doc2data("menu"));
        }
      });
    const order_detacher = db
      .doc(`restaurants/${this.restaurantId()}/orders/${this.orderId}`)
      .onSnapshot(
        order => {
          if (order.exists) {
            const order_data = order.data();
            this.orderInfo = order_data;
          } else if (!this.isDeleting) {
            this.notFound = true;
          }
        },
        error => {
          // Because of the firestore.rules, it causes "insufficient permissions"
          // if the order does not exist.
          console.log(error);
          this.notFound = true;
        }
      );
    this.detacher = [restaurant_detacher, menu_detacher, order_detacher];
  },
  destroyed() {
    if (this.detacher) {
      this.detacher.map(detacher => {
        detacher();
      });
    }
  },
  computed: {
    hidePayment() {
      return releaseConfig.hidePayment;
    },
    orderName() {
      return nameOfOrder(this.orderInfo);
    },
    orderStatusKey() {
      return Object.keys(order_status).reduce((result, key) => {
        return order_status[key] === this.orderInfo.status ? key : result;
      }, "unexpected");
    },
    newOrder() {
      return this.orderInfo.status === order_status.new_order;
    },
    just_validated() {
      return this.orderInfo.status === order_status.validation_ok;
    },
    just_paid() {
      return this.orderInfo.status === order_status.customer_paid;
    },
    paid() {
      return this.orderInfo.status >= order_status.customer_paid;
    },
    orderItems() {
      if (this.menus.length > 0 && this.orderInfo.order) {
        return Object.keys(this.orderInfo.order).map(key => {
          const num = this.orderInfo.order[key];
          return {
            item: this.menuObj[key],
            count: num,
            id: key,
            specialRequest: this.specialRequest(key)
          };
        });
      }
      return [];
    },
    menuObj() {
      return this.array2obj(this.menus);
    },
    orderId() {
      return this.$route.params.orderId;
    }
  },
  methods: {
    handleTipChange(tip) {
      console.log("handleTipChange", tip);
      this.tip = tip;
    },
    handleCardStateChange(state) {
      this.cardState = state;
    },
    specialRequest(key) {
      const option = this.orderInfo.options && this.orderInfo.options[key];
      if (option) {
        return option.filter(choice => choice).join(", ");
      }
      return "";
    },
    async handleEditItems() {
      try {
        this.isDeleting = true;
        await db
          .doc(`restaurants/${this.restaurantId()}/orders/${this.orderId}`)
          .delete();
        console.log("suceeded");
      } catch (error) {
        this.isDeleting = false;
        console.log("failed");
      }
      this.$router.push({
        path: `/r/${this.restaurantId()}#${this.orderId}`
      });
    },
    async handlePayment() {
      this.isPaying = true;
      const {
        error,
        paymentMethod
      } = await this.$refs.stripe.createPaymentMethod();

      if (error) {
        this.isPaying = false;
        console.log(error);
        return;
      }

      try {
        const { data } = await checkoutCreate({
          paymentMethodId: paymentMethod.id,
          restaurantId: this.restaurantId(),
          orderId: this.orderId
        });
        // const result = await checkoutConfirm({
        //   paymentIntentId: data.result.paymentIntentId,
        //   orderPath: `restaurants/${this.restaurantId()}/orders/${this.orderId}`
        // });
        window.scrollTo(0, 0);
      } catch (error) {
        console.error(error);
      } finally {
        this.isPaying = false;
      }
    },
    async handleNoPayment() {
      try {
        // HACK: Workaround until we implement sprite
        await db
          .doc(`restaurants/${this.restaurantId()}/orders/${this.orderId}`)
          .update({
            status: order_status.customer_paid,
            timePaid: firestore.FieldValue.serverTimestamp()
          });
        console.log("suceeded");
        window.scrollTo(0, 0);
      } catch (error) {
        console.log("failed", error);
      }
    },
    async handleCancelPayment() {
      console.log(this.orderInfo.result);

      try {
        this.isCanceling = true;
        const { data } = await checkoutCancel({
          paymentIntentId: this.orderInfo.result.id,
          orderPath: `restaurants/${this.restaurantId()}/orders/${this.orderId}`
        });
      } catch (error) {
        // BUGBUG: Implement the error handling code here
        console.error(error);
      } finally {
        this.isCanceling = false;
      }
    }
  }
};
</script>
<style lang="scss" scoped>
.tax {
  margin-top: -2rem !important;
}
.thankyou {
  color: $primary;
  font-size: 1.2em;
  font-weight: bold;
  margin-bottom: 0.5rem;
}
.thankyou-icon {
  color: $primary;
  font-size: 8em;
  margin: 1rem;
}
</style>