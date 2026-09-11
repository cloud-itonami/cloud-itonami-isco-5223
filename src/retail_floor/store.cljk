(ns retail-floor.store
  "SSoT for the ISCO-08 5223 independent retail-floor-sales sole-proprietor
  actor, behind a `Store` protocol so the backend is a swap (MemStore
  default ‖ a real Datomic/kotoba-server backend, per the itonami actor
  pattern).

  Domain = independent retail floor sales practice:

    sku            — a stocked product (skuId, name, listPriceCents)
    price-change   — a price adjustment for a sku (changeId, skuId,
                     newPriceCents)
    sale           — a completed sale transaction (saleId, skuId, qty,
                     amountCents)

  The append-only records are the operating ledger: a price-change or sale
  must reference a registered sku, and price-changes/sales are never
  mutated in place, only appended.")

(defprotocol Store
  (sku [st sku-id])
  (price-changes-of [st sku-id])
  (sales-of [st sku-id])
  (register-sku! [st sku])
  (record-price-change! [st price-change])
  (record-sale! [st sale]))

(defrecord MemStore [state]
  Store
  (sku [_ sku-id]
    (get-in @state [:skus sku-id]))
  (price-changes-of [_ sku-id]
    (filter #(= sku-id (:sku-id %)) (:price-changes @state)))
  (sales-of [_ sku-id]
    (filter #(= sku-id (:sku-id %)) (:sales @state)))
  (register-sku! [_ sku]
    (swap! state assoc-in [:skus (:sku-id sku)] sku))
  (record-price-change! [_ price-change]
    (swap! state update :price-changes (fnil conj []) price-change))
  (record-sale! [_ sale]
    (swap! state update :sales (fnil conj []) sale)))

(defn mem-store
  ([] (mem-store {}))
  ([seed]
   (->MemStore (atom (merge {:skus {} :price-changes [] :sales []} seed)))))
