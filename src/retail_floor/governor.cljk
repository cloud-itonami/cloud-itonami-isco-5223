(ns retail-floor.governor
  "RetailFloorGovernor — the independent safety/traceability layer for the
  ISCO-08 5223 independent retail-floor-sales actor. The Sales Advisor
  proposes actions (price-change, sale); it has no notion of sku
  provenance or discount-magnitude risk, so this MUST be a separate system
  able to *reject* a proposal and fall back to HOLD — the itonami-actor
  pattern (independent Governor gates a proposing actor) applied to this
  occupation.

  Charter (mirrors ADR-2607011000 robotics premise + ADR-2607012000
  cloud-itonami-isco): the actor never dispatches a robot action or writes an
  operating record the governor refuses. A price-change that discounts a
  sku by more than `discount-threshold` from its list price ALWAYS
  requires human sign-off — it can never be auto-approved.

  HARD invariants for :retail/propose:
    1. SKU provenance   — a price-change or sale must reference a
       registered sku.
    2. No-actuation     — the proposal must not directly mutate a
       price-change or sale record outside the record-price-change!/
       record-sale! path (effect must be :propose, never a raw store
       write).
    3. Discount safety  — a price-change whose new price is more than
       `discount-threshold` below the sku's list price always requires
       :high or higher safety-class, forcing human sign-off; it is never
       auto-approved regardless of confidence.
  SOFT:
    4. Confidence floor → escalate."
  (:require [retail-floor.store :as store]))

(def confidence-floor 0.6)
(def discount-threshold 0.30)
(def safety-classes [:none :low :medium :high :safety-critical])

(defn- safety-rank [safety-class]
  (let [idx (.indexOf safety-classes safety-class)]
    (if (neg? idx) 0 idx)))

(defn- deep-discount? [list-price-cents proposal]
  (and (= :price-change (:action proposal))
       (number? list-price-cents)
       (pos? list-price-cents)
       (number? (:new-price-cents proposal))
       (> (- 1 (/ (:new-price-cents proposal) (double list-price-cents)))
          discount-threshold)))

(defn- hard-violations [{:keys [sku-fn]} proposal]
  (let [{:keys [sku-id safety-class effect]} proposal
        found-sku (sku-fn sku-id)]
    (cond-> []
      (nil? found-sku)
      (conj {:rule :no-sku :detail (str "未登録 sku " sku-id)})

      (not= :propose effect)
      (conj {:rule :no-actuation :detail "effect は :propose のみ許可（直接書込禁止）"})

      (and found-sku
           (deep-discount? (:list-price-cents found-sku) proposal)
           (< (safety-rank (or safety-class :none)) (safety-rank :high)))
      (conj {:rule :discount-safety
             :detail (str "list price から discount-threshold("
                           discount-threshold ") を超える値下げ — :high 以上の safety-class が必須")}))))

(defn assess
  "Assess a proposal against `env` (a map with `:sku-fn` lookup, decoupled
  from any concrete Store so this stays pure). Returns
  `{:decision :proceed|:hold|:human-approval :violations [...] :confidence n}`."
  [env proposal]
  (let [violations (hard-violations env proposal)
        safety-class (or (:safety-class proposal) :none)
        confidence (or (:confidence proposal) 0.0)]
    (cond
      (seq violations)
      {:decision :hold :violations violations :confidence confidence}

      (>= (safety-rank safety-class) (safety-rank :high))
      {:decision :human-approval :violations [] :confidence confidence}

      (< confidence confidence-floor)
      {:decision :human-approval :violations [] :confidence confidence
       :reason :low-confidence}

      :else
      {:decision :proceed :violations [] :confidence confidence})))

(defn env-for-store
  "Build the decoupled env map `assess` needs from a concrete
  `retail-floor.store/Store` implementation."
  [store]
  {:sku-fn #(store/sku store %)})
