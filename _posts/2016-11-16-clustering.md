---
layout: post
title:  "SciPyで階層的クラスタリングしてみた"
categories: python
---

SciPyで階層的クラスタリングをしてみたので、そのメモです。

### 利用例

Google Cloud Vision APIで文字認識した結果をクラスタリングにかけてみました。
クラスタリングを使えば、文章の見た目のまとまりを認識でき、認識結果の使い勝手がよくなりそうです。

たとえば、歌詞カードの文字認識をしたいとします。
歌詞カードでは次のように、横書きの文章が1ページに並べられていることはよくあります。

```
       赤とんぼ                          夕焼け小焼け

  夕焼小焼の赤とんぼ                 夕焼け小焼けで日が暮れて
負われて見たのはいつの日か              山のお寺の鐘がなる
   山の畑の桑の実を                 おててつないでみなかえろう
 小籠に摘んだはまぼろしか          からすといっしょにかえりましょ
  十五で姐やは嫁に行き                子供がかえったあとからは
 お里のたよりも絶えはてた              まるい大きなお月さま
   夕焼小焼の赤とんぼ                  小鳥が夢を見るころは
  とまっているよ竿の先                 空にはきらきら金の星
```

これを文字認識したとき、我々の期待は当然、「赤とんぼ」と「夕焼け小焼け」の歌詞をそれぞれ読みとれることです。

しかし、Google Cloud Vision APIを使ったとき、素直にはいきません。

Google Cloud Vision APIの認識結果は単語とその座標の組のリストになっていますが、このリスト上の順番は、左上から右下へ向けて読んだ順番になるからです。
つまり、認識結果のリストでは、「赤とんぼ」、「夕焼け小焼け」、「夕焼小焼の赤とんぼ」、「夕焼け小焼けで日が暮れて」、「負われて見たのはいつの日か」、「山のお寺の鐘がなる」というように、2つの歌詞が混ざってしまいます。

無論、これはGoogle Cloud Vision APIに問題があるのではなく、むしろ、文章を一目見れば、その配置から意味を無意識に抽出できる我々の目がすごいんだと言えるでしょう。

ここで、Google Cloud Vision APIの出力では単語の座標もわかるので、それに基づいてクラスタリングすれば、なんかうまくできそうな気がしました。

### クラスタリングの仕方

SciPy関係で使ったのは、以下の関数だけです。

#### [scipy.spatial.distance.pdist](https://docs.scipy.org/doc/scipy-0.18.1/reference/generated/scipy.spatial.distance.pdist.html)

`p`次元空間上のN個の座標リストを渡すと、距離行列(各点間の距離のリスト)を返してくれます。

2次元座標を2個とか3個とか渡したときの実行結果は次のようになります。:

```
>>> pdist([[1,1], [1,2]])
array([ 1.])
>>> pdist([[1,1], [1,2], [1,4]])
array([ 1.,  3.,  2.])
```

#### [scipy.cluster.hierarchy.ward](https://docs.scipy.org/doc/scipy-0.18.1/reference/generated/scipy.cluster.hierarchy.ward.html#scipy.cluster.hierarchy.ward)

ウォード法に従ってクラスタリングしてくれます。
入力は距離行列(`pdist`の出力結果)で、出力はクラスタリング結果です。

クラスタリング結果のフォーマットは、`linkage`の出力と同じだそうです。

[scipy.cluster.hierarchy.linkage](https://docs.scipy.org/doc/scipy-0.18.1/reference/generated/scipy.cluster.hierarchy.linkage.html#scipy.cluster.hierarchy.linkage)

> An (n−1)(n−1) by 4 matrix Z is returned. At the ii-th iteration, clusters with indices Z[i, 0] and Z[i, 1] are combined to form cluster n+in+i. A cluster with an index less than nn corresponds to one of the nn original observations. The distance between clusters Z[i, 0] and Z[i, 1] is given by Z[i, 2]. The fourth value Z[i, 3] represents the number of original observations in the newly formed cluster.

どういうことかというと、実行結果はこんな感じです。

```
>>> ds = pdist([[1,1], [1,2], [1,4]])
>>> ward(ds)
array([[ 0.        ,  1.        ,  1.        ,  2.        ],
       [ 2.        ,  3.        ,  2.88675135,  3.        ]])

```

出力されたリストは、1行がクラスタの1階層になっており、4つの値で表現されます。
- 1列目、2列目は、クラスタのノードの番号、
- 3列目はノード間の距離、
- 4列目はつくられるクラスタの合計ノード数です。

次のように読みます。

1行目では、ノード0とノード1でクラスタをつくり、その距離は1になります。
ここでつくられた新しいクラスタはノード3とします(はじめに与えたノードが3つだったので、新しいノードを4つ目と考える)。

```
[ 0.        ,  1.        ,  1.        ,  2.        ]
```

2行目では、ノード2とノード3でクラスタをつくり、その距離は2.887になります。

```
[ 2.        ,  3.        ,  2.88675135,  3.        ]
```

今回は、これで全てのノードをまとめられたので、これで完了です。

### サンプルプログラム

正直、このサンプルプログラムは書いている途中で飽きました。
[kikei/nirvanam-watch](https://github.com/kikei/nirvanam-watch)で実用しているので、そちらを参照してください。

```python
import requests
import json
import base64

GOOGLE_CLOUD_VISION_API_URL = 'https://vision.googleapis.com/v1/images:annotate?key='

def load_image(image_path):
    bs = None
    with open(image_path, 'rb') as f:
        bs = base64.b64encode(f.read())
    return bs

def make_request_body(image_content):
    req_body = json.dumps({
        'requests': [{
            'image': {
                'content': image_content.decode('utf-8')
            },
            'features': [{
                'type': 'TEXT_DETECTION',
                'maxResults': 100
            }]
        }]
    })
    return req_body

def detect_text(google_api_key, imagefile):
    image_content = load_image(imagefile)
	
    api_uri = GOOGLE_CLOUD_VISION_API_URL + google_api_key
    req_body = make_request_body(image_content)
    res = requests.post(api_uri, data=req_body)
    if not res.ok:
        return None
    return res.json()

def clusterize(annotations):
    from scipy.spatial.distance import pdist
    from scipy.cluster.hierarchy import ward, dendrogram
    # from matplotlib.pyplot import show
        
    def calc_center(annotation):
        vs = annotation['boundingPoly']['vertices']
        return [ (vs[0]['x'] + vs[2]['x']) / 2.0 /
                  CLUSTERING_HORIZONTAL_PRIORITY,
                 (vs[0]['y'] + vs[2]['y']) / 2.0]
    descs = list(map(calc_center, annotations))

    dists = pdist(descs)
    clusters = ward(dists)
        
    # print('drawing dendrogram')
    # dendrogram(clusters)
    # show()
    return clusters

if __name__ == '__main__':
    imagefile = os.path.join('.', 'songs.png')
    google_api_key = 'hoge'
    
    data = detect_text(imagefile, google_api_key)
    annotations = data['responses'][0]['textAnnotations']
    annotations = annotations[1:]
    print(clusterize(annotations))
```

### まとめ

備忘録的に書いているつもりだったが、途中で飽きた。
