# 趨勢科技 - TBrain - 無檔案的惡意程式辨識

## 結果
Public Leaderboard： 第十六名， AUC 為 0.953174 。

Private Leaderboard： 第十一名， AUC 為 0.962957 。

Top 3%： 11/424 。

## 前處理
先透過 to_fileID.ipynb 將資料轉換成一個 File 一個檔案。

## 特徵
第一類的特徵是觀察並思考檔案的 Log ，怎麼樣能夠判別是否為惡意程式。比較有用的特徵如：該檔案每個使用者 Log 數量的平均、
該檔案每次 Log 間隔時間的中位數、該檔案第一個二十四小時內 Log 的數量...等等。

接下來組員有提到他們用中毒率來當作特徵，發現是個滿好的特徵。如使用者的中毒率為：使用者中毒的檔案數 / 使用者所有的檔案數。
可以想像成是 P(file_is_virus|user) 。這邊使用了使用者以及產品的中毒率。

不過使用中毒率特徵要注意的是，由於是透過先看答案所算出來的特徵，不能夠拿所有的 Training Set 下去統計，否則等於作弊，會 Overfitting 。
因此我將 Training Set 切成五份，每次都拿四份做統計，來產生另一份的特徵（也就是說我們並沒有先看到這份資料的答案來產生特徵），
最後再將五份合併成完整的 Training Set 。

另外由於為時間序列的資料，所以可能後來會有新的使用者，並沒有中毒率。因此最後有加一些特徵是關於，該檔案的使用者過去是否有足夠的 Log 紀錄。
希望模型能夠判斷是否要使用中毒率的特徵。

## 訓練
基本上使用 LightGBM 來訓練。第一次訓練拿所有的特徵，主要拿來觀察哪些特徵比較重要。接著挑出重要的特徵做第二次的訓練當作最後的模型。
在我自己的模擬中，最好的 LightGBM 模型只比最後送出的模型 AUC 少了 0.004 。

在最後的時候另外加上了 NN 的模型，還不是很熟習 Tensorflow ，拿 Scikit-learn 的 NN 來用。最好的模型效果和 LightGBM 差不多，
但是似乎多了些 Overfitting 。因此 NN 的模型是將 Training Set 每次拿 4/5 訓練，總共訓練出五個模型後，取平均當作最後的模型。
透過這樣簡單的 Ensemble 降低 Variance 。

LightGBM * 0.6 + NN * 0.4 為送出的結果。

完整從產生特徵到訓練的檔案為 tm-competition-1.ipynb 。

## 其他題外話
Google BigQuery 滿好用的，拿來用 SQL 產生每個使用者的中毒率等等，都只需要幾秒鐘的時間。
選擇 Destination Table ，並且打勾 Allow Large Results ，可以輸出較大的結果。

當資料量很大， Pandas 在做 String 比對會變得相當慢。我用兩種方法加速，
第一種是將要比對的欄位做 groupby ，之後用 get_group 取出資訊，好像是因為有做 cache 所以速度變快，
但第一種方式在更大的資料時，還是沒有辦法。

第二種是將要比對的欄位做輸出成 list 並 sort ，因此就可以用 bisect 做 binary search ，找到 index 。
要注意的是， DataFrame 記得也要 sort_values 該欄位，並且重新 reset_index(drop=True) 或用 iloc 取值，讓兩邊的 index 順序是一致的。

