---
layout: post
title:  "javascript로 빙고판 그리기"
description: javascript , jquery 를 사용하여 빙고판을 그리고 빙고가 완성된 줄수를 체크 하는 기능 만들기  
date:   2020-05-19 00:00:00 +000
categories: javascript jquery 
---

빙고 줄수를 n 입력하면 숫자 1부터 n의 제곱까지의 숫자를 활용한 빙고판을 그려주고<br>
빙고판 영역을 클릭할 경우 선택구분 표시 및 현재의 빙고 수를 한번 보여 주는 <br>
스크립트를 만들어 보려고 합니다. 

# 웹 빙고게임 틀 만들기 

```css
    .b_table {
    display: table;
    }

    .b_table_row {
    display: table-row;
    }

    .b_table_cell {
    width:30px;
    height:30px;
    display: table-cell;
    border: 1px solid gray;
    cursor: pointer;
    vertical-align: middle;
    text-align:center;
    }
```

```html
    빙고판 줄수 : <input type="text" id="num" value="5"><br>
    <div >현재 빙고 수 :  <span class="cnt"></span></div>
    <a href="javascript:void(0)" onclick="$.bingoJs.ready($('#num').val());return false;">빙고시작</a>

    <div id="bingo"></div>
```
위의 내용은 기본적인 빙고판 시작을 위한 html 과 빙고판 모양 유지를 위한 css 내용입니다. 

```javascript
    ;(function($) {
        var option = {
            size: 3,
            list: [],
            id: 'bingo'
        }
        var bingox = {}
        var bingoy = {}
        var bingoz = [0,0]
        var bingcnt = 0
        $.bingoJs = {
            reset:function(){
                bingox = {}
                bingoy = {}
                bingoz = [0,0]
                bingcnt = 0
                option = {
                    size: 3,
                    list: [],
                    id: 'bingo'
                }
            },
            ready: function(s, id) {
                this.reset()
                if (s) option.size = parseInt(s)
                if (id) option.id = id
                for (var i = 1; i <= (option.size * option.size); i++) option.list.push(i)
                this.shuffle()
                //초기화
                $("#" + option.id).html("")
                //틀추가
                $("#" + option.id).append("<div class='b_table'></div>")
                var _row = ""
                for (var i = 0; i < (option.size * option.size); i++) {
                    if (i % option.size === 0) {
                        _row += "<div class='b_table_row'>"
                    }
                    _row += "<div class='b_table_cell cell_" + i + "' onclick='$.bingoJs.clickcell(" + i + ")'  data-selected=false >" + option.list[i] + "</div>";
                    if (i % option.size === (option.size - 1)) {
                        _row += "</div>"
                        $(".b_table").append(_row)
                        _row = ""
                    }
                }
                this.chkbingocnt()
            },
            shuffle: function() {
                for (var j, x, i = option.list.length; i; j = parseInt(Math.random() * i), x = option.list[--i], option.list[i] = option.list[j], option.list[j] = x);
            },
            clickcell: function(s) {
                if($(".cell_"+s).get(0) && $(".cell_"+s).data("selected") === false ){
                var c = s + 1
                var x = (c % option.size === 0) ? option.size : c % option.size
                var y = Math.floor(((c + option.size) - 1) / option.size)

                if (!bingox[x]) bingox[x] = 0
                if (!bingoy[y]) bingoy[y] = 0
                bingox[x]++
                bingoy[y]++
                if (bingox[x] == option.size) bingcnt++
                if (bingoy[y] == option.size) bingcnt++

                if(x === y){
                    bingoz[0]++
                    if (bingoz[0] == option.size) bingcnt++
                }

                if((x+y) === (option.size+1)){
                    bingoz[1]++
                    if (bingoz[1] == option.size) bingcnt++
                }
                $('.cell_' + s).data("selected",true)
                $('.cell_' + s).css("background-color", "red")
                this.chkbingocnt()
                }            
            },
            chkbingocnt: function() {
                $(".cnt").html(bingcnt)
            }
        }
    })(jQuery);

```
위의 스크립트 내용은 스크립트를 조금만 보실수 있다면 구조 보시는데 문제가 없다고 생각하기에 별도의 설명은 없습니다. <Br>

소스 보기 : <a href="https://jsfiddle.net/chodaeyun/xp2hr6v1/" target="_blank">https://jsfiddle.net/chodaeyun/xp2hr6v1/</a><br>

빙고 최대의 경우의 수 : n*2+2 <br>
가로 빙고 : x <br>
세로 빙고 : y <br>
대각선 빙고 : z <br>
reset : 변수 초기화 <br>
ready : 빙고 시작 <br>
clickcell : 숫자 클릭 시 선택 전환 및 빙고 체크 <br>
chkbingocnt : 현재 빙고가 완성된 줄수 표기 호출 <br>
shuffle : 1~n 숫자 순서 랜덤 적용 <br>
