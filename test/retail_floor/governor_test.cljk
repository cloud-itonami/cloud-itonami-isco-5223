(ns retail-floor.governor-test
  (:require [clojure.test :refer [deftest is testing]]
            [retail-floor.store :as store]
            [retail-floor.governor :as governor]))

(defn- fresh-store []
  (let [st (store/mem-store)]
    (store/register-sku! st {:sku-id "sku-1" :name "widget" :list-price-cents 1000})
    st))

(deftest proceeds-on-clean-sale
  (let [st (fresh-store)
        env (governor/env-for-store st)
        proposal {:action :sale :sku-id "sku-1" :safety-class :low
                   :effect :propose :confidence 0.9}]
    (is (= :proceed (:decision (governor/assess env proposal))))))

(deftest holds-on-unregistered-sku
  (let [st (fresh-store)
        env (governor/env-for-store st)
        proposal {:action :sale :sku-id "no-such-sku" :safety-class :low
                   :effect :propose :confidence 0.9}
        result (governor/assess env proposal)]
    (is (= :hold (:decision result)))
    (is (some #(= :no-sku (:rule %)) (:violations result)))))

(deftest holds-on-no-actuation-violation
  (let [st (fresh-store)
        env (governor/env-for-store st)
        proposal {:action :sale :sku-id "sku-1" :safety-class :low
                   :effect :direct-write :confidence 0.9}
        result (governor/assess env proposal)]
    (is (= :hold (:decision result)))
    (is (some #(= :no-actuation (:rule %)) (:violations result)))))

(deftest proceeds-on-small-discount
  (let [st (fresh-store)
        env (governor/env-for-store st)
        proposal {:action :price-change :sku-id "sku-1" :new-price-cents 900
                   :safety-class :low :effect :propose :confidence 0.9}]
    (is (= :proceed (:decision (governor/assess env proposal))))))

(deftest holds-on-deep-discount-without-high-safety-class
  (let [st (fresh-store)
        env (governor/env-for-store st)
        proposal {:action :price-change :sku-id "sku-1" :new-price-cents 500
                   :safety-class :medium :effect :propose :confidence 0.9}
        result (governor/assess env proposal)]
    (is (= :hold (:decision result)))
    (is (some #(= :discount-safety (:rule %)) (:violations result)))))

(deftest human-approval-on-deep-discount-with-high-safety-class
  (let [st (fresh-store)
        env (governor/env-for-store st)
        proposal {:action :price-change :sku-id "sku-1" :new-price-cents 500
                   :safety-class :high :effect :propose :confidence 0.9}]
    (is (= :human-approval (:decision (governor/assess env proposal))))))

(deftest human-approval-on-low-confidence
  (let [st (fresh-store)
        env (governor/env-for-store st)
        proposal {:action :sale :sku-id "sku-1" :safety-class :none
                   :effect :propose :confidence 0.2}
        result (governor/assess env proposal)]
    (is (= :human-approval (:decision result)))
    (is (= :low-confidence (:reason result)))))

(deftest store-records-append-only
  (let [st (fresh-store)]
    (store/record-sale! st {:sale-id "s1" :sku-id "sku-1" :qty 2 :amount-cents 2000})
    (store/record-price-change! st {:change-id "c1" :sku-id "sku-1" :new-price-cents 950})
    (is (= 1 (count (store/sales-of st "sku-1"))))
    (is (= 1 (count (store/price-changes-of st "sku-1"))))))

(deftest a-proposal-without-confidence-does-not-proceed
  (testing "確信度を言っていない提案は、確信していると言っていないので auto-proceed
            させない。この既定は 2026-07-30 まで 1.0 で、:confidence を持たない提案が
            :proceed していた（ADR-2607309100）。fleet の boolean 方言 346 件はすべて
            0.0 既定で、うち isco-5419 はそれを明示的にテストしている。"
    (let [st (fresh-store)
          env (governor/env-for-store st)
          proposal {:action :sale :sku-id "sku-1" :safety-class :low :effect :propose}
          result (governor/assess env proposal)]
      (is (= 0.0 (:confidence result))
          "欠落した :confidence は 0.0 であって 1.0 ではない")
      (is (not= :proceed (:decision result))
          "確信度不明の提案が自動で通ってはならない"))))
