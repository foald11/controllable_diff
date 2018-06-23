# controllable-diff
## 概要
ある２つのデータ配列の差分を算出することができます．
差分の計算は，エディットグラフの最短経路を計算することで行います．

通常のdiffは，２つの文字列を比較する用途で用いられますが，
本ツールは，以下の機能により，複雑なデータ構造の差分をお好みの粒度で算出できます．
- ２要素の比較方法を変更できる
- 算出した差分の結果の後処理を変更できる

## 使用用途
本ツールは，以下のような用途で使用することができます．
- 辞書オブジェクトのような文字列以外のデータ配列で差分を抽出したい
  - JSON形式のログの比較など
- ログに含まれるタイムスタンプを無視して差分を抽出したい
  - ログに含まれるタイムスタンプの誤差を吸収して差分を抽出したい
- 形式の異なる２つのデータ配列から，特定のパラメータのみ差分を抽出したい
- 算出した差分を他の解析に利用したい
- 算出した差分から，一致した部分や片方にだけ含まれていた部分のみを抽出したい

など

## 使用方法
diff.pyで宣言されたdiffメソッドを呼び出すことで利用できます．
diffメソッドは，比較したい２つのデータ配列以外に２つのコールバック関数を引数に渡す必要があります．

~~~~
# arr1 : 比較対象１
# arr2 : 比較対象２
# comp_func : 比較用コールバック関数
# result_func : 後処理用コールバック関数
# 返り値 : result_funcコールバック関数の返り値と同じ値が返る

diff(arr1,arr2,comp_func,result_func)
~~~~

### コールバック関数
#### comp_func (第３引数)
comp_funcコールバック関数は，arr1とarr2からそれぞれ取り出された１要素の比較処理を定義しておく必要があります．
comp_funcコールバック関数は，以下の仕様を満たしている必要があります．

~~~~
# a : arr1の１要素
# b : arr2の１要素
# 返り値 : aとbが一致しているならTrue，一致していなければFalse

bool comp_func(a,b)
~~~~

diff.pyには，comp_funcコールバック関数のデフォルト実装としてdefault_compareメソッドが用意されています．
default_compareメソッドは，aとbをイコール演算子で単純に比較します．

#### result_func (第４引数)
result_funcコールバック関数は，差分が抽出された後の後処理を定義できます．
データ形式に合わせて比較結果を解析したり，比較結果を表示したりすることができます．
result_funcコールバック関数は，以下の仕様を満たしている必要があります．

~~~~
#result_list : 差分の抽出結果
#arr1 : 比較対象１(diffメソッドのarr1と同じ)
#arr2 : 比較対象２(diffメソッドのarr2と同じ)
#返り値 : 任意のオブジェクト(diffメソッドの返り値になります) 

Object result_func(result_list, arr1, arr2)
~~~~

result_listは，エディットグラフから導出した最短経路を構成する*ノードの配列*です．
各ノード(Node)には以下のような属性があり，arr1とarr2と照らし合わせながら差分結果を解析することができます．
- Node.dir
  - エディットグラフでどの方向に遷移してきたかを示します
  - 's'の時，この要素は，arr1とarr2で一致した要素です．
  - 'b'の時，この要素は，arr1にのみ含まれていた要素です．
  - 'r'の時，この要素は，arr2にのみ含まれていた要素です．
- Node.mi
  - このノードで比較したarr1の要素のインデックス値を示します．
  - node.dirが'b'の時，arr1[node.mi]とすることでarr1にのみ含まれていた要素のデータを参照できます．
- Node.ni
  - このノードで比較したarr2の要素のインデックス値を示します．
  - node.dirが'r'の時，arr2[node.ni]とすることでarr2にのみ含まれていた要素のデータを参照できます．

diff.pyには，result_funcコールバック関数のデフォルト実装としてdefault_print_resultメソッドが用意されています．
default_print_resultメソッドは，差分の抽出結果を色付きのテキストで表示し，最後にサマリを表示します．

## 使用例
~~~~
>>> import diff
>>>
>>> diff.diff("test", "tea time", diff.default_compare, diff.default_print_result)
   t
   e
 + a
 +
 - s
   t
 + i
 + m
 + e

summary: match=3, add=5, remove=1
0
>>> 
~~~~

~~~~
>>> import diff
>>>
>>> str1 = ["Hello,", "My Name is", "foald11"]
>>> str2 = ["Hello,", "My Nicname is", "foald11"]
>>> 
>>> diff.diff(str1, str2, diff.default_compare, diff.default_print_result)
   Hello,
 + My Nicname is
 - My Name is
   foald11

summary: match=2, add=1, remove=1
0
>>>
~~~~

~~~~
>>> import diff
>>>
>>> log1 = [ {"time": "14:00.00" , "msg": "Login Failed: root"}, ]
>>> log1.append( {"time": "14:00.01" , "msg": "Login Failed: John"} )
>>> log1.append( {"time": "14:00.02" , "msg": "Login Failed: Michel"} )
>>> log1.append( {"time": "14:00.03" , "msg": "Login Success: Admin"} )
>>> log1.append( {"time": "14:01.02" , "msg": "Welcom!!"} )
>>>
>>> log2 = [ {"time": "23:58.30" , "msg": "Login Failed: root"}, ]
>>> log2.append( {"time": "23:58.31" , "msg": "Login Failed: Michel"} )
>>> log2.append( {"time": "23:58.32" , "msg": "Login Failed: Bob"} )
>>> log2.append( {"time": "23:58.33" , "msg": "Login Success: Admin"} )
>>> log2.append( {"time": "23:59.30" , "msg": "Welcom!!"} )
>>>
>>> def compare_log(a,b):
...     if a["msg"] == b["msg"]:
...             return True
...     return False
...
>>> 
>>> diff.diff(log1, log2, compare_log, diff.default_print_result)
   {'msg': 'Login Failed: root', 'time': '23:58.30'}
 - {'msg': 'Login Failed: John', 'time': '14:00.01'}
   {'msg': 'Login Failed: Michel', 'time': '23:58.31'}
 + {'msg': 'Login Failed: Bob', 'time': '23:58.32'}
   {'msg': 'Login Success: Admin', 'time': '23:58.33'}
   {'msg': 'Welcom!!', 'time': '23:59.30'}

summary: match=4, add=1, remove=1
0
>>>
~~~~

## TODO
エディットグラフの最短経路導出に，最適化の余地が多いにあります．
