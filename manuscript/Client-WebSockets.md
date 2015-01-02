## WebSocket Communication 

The ````birdwatch.communicator```` **[namespace](https://github.com/matthiasn/BirdWatch/blob/574d2178be6f399086ad2a5ec35c200d252bf887/Clojure-Websockets/MainApp/src/cljs/birdwatch/communicator.cljs)** handles the interaction with the server-side application through the use of a WebSocket connection provided by the **[sente](https://github.com/ptaoussanis/sente)** library.

~~~
(ns birdwatch.communicator
  (:require-macros [cljs.core.match.macros :refer (match)]
                   [cljs.core.async.macros :refer [go-loop go]])
  (:require [birdwatch.channels :as c]
            [birdwatch.util :as util]
            [birdwatch.state :as state]
            [cljs.core.match]
            [taoensso.sente  :as sente  :refer (cb-success?)]
            [taoensso.sente.packers.transit :as sente-transit]
            [cljs.core.async :as async :refer [<! >! chan put! alts! timeout]]))

(enable-console-print!)

(def packer
  "Defines our packing (serialization) format for client<->server comms."
  (sente-transit/get-flexi-packer :json))

(let [{:keys [chsk ch-recv send-fn state]} (sente/make-channel-socket! "/chsk" {:packer packer :type :auto})]
  (def chsk       chsk)
  (def ch-chsk    ch-recv) ; ChannelSocket's receive channel
  (def chsk-send! send-fn) ; ChannelSocket's send API fn
  (def chsk-state state))  ; Watchable, read-only atom

(defn query-string
  "format and modify query string"
  []
  {:query_string {:default_field "text"
                  :default_operator "AND"
                  :query (str "(" (:search @state/app) ") AND lang:en")}})

(defn start-percolator
  "trigger starting of percolation matching of new tweets"
  []
  (chsk-send! [:cmd/percolate {:query (query-string)
                               :uid (:uid @chsk-state)}]))

(def prev-chunks-loaded (atom 0))

(defn load-prev
  "load previous tweets matching the current search"
  []
  (let [chunks-to-load 10
        chunk-size 500]
    (when (< @prev-chunks-loaded chunks-to-load)
      (chsk-send! [:cmd/query {:query (query-string)
                               :n chunk-size
                               :uid (:uid @chsk-state)
                               :from (* chunk-size @prev-chunks-loaded)}])
      (swap! prev-chunks-loaded inc))))

(defn start-search
  "initiate new search by starting SSE stream"
  []
  (let [search (:search-text @state/app)
        s (if (= search "") "*" search)]
    (reset! state/app (state/initial-state))
    (reset! prev-chunks-loaded 0)
    (swap! state/app assoc :search-text search)
    (swap! state/app assoc :search s)
    (aset js/window "location" "hash" (js/encodeURIComponent s))
    (start-percolator)
    (dotimes [n 4] (load-prev))))

(defn- event-handler [{:keys [event]}]
  (match event
         [:chsk/state {:first-open? true}] (do (print "Socket established!") (start-search))
         [:chsk/state new-state]           (print "Chsk state change:" new-state)
         [:chsk/recv  payload]
         (let [[msg-type msg] payload]
           (match [msg-type msg]
                  [:tweet/new             tweet] (put! c/tweets-chan tweet)
                  [:tweet/missing-tweet   tweet] (put! c/missing-tweet-found-chan tweet)
                  [:tweet/prev-chunk prev-chunk] (do (put! c/prev-chunks-chan prev-chunk)(load-prev))
                  [:stats/users-count        uc] (put! c/user-count-chan uc)
                  [:stats/total-tweet-count ttc] (put! c/total-tweets-count-chan ttc)))
         :else (print "Unmatched event: %s" event)))

(defonce chsk-router (sente/start-chsk-router! ch-chsk event-handler))

; loop for sending messages about missing tweet to server
(go-loop [] (let [tid (<! c/tweet-missing-chan)]
              (chsk-send! [:cmd/missing {:id_str tid :uid (:uid @chsk-state)}])
              (recur)))

(defn ^:export send-state
  "helper function to send state to server (where it can be pretty printed for debugging)"
  []
  (chsk-send! [:some/state {:by-followers (into {} (:by-followers @state/app))
                            :by-retweets  (into {} (:by-followers @state/app))
                            :by-favorites  (into {} (:by-favorites @state/app))
                            :by-rt-since-startup  (into {} (:by-rt-since-startup @state/app))
                            :by-reach  (into {} (:by-reach @state/app))
                            :by-id  (into {} (:by-id @state/app))}]))
~~~

