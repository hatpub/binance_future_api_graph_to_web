import plotly.graph_objects as go
from datetime import datetime
from dash import Dash, callback_context
from dash_auth import BasicAuth
from dash_table import DataTable
import dash_core_components as dcc
import dash_html_components as html
from dash.dependencies import Input, Output, State
import pymysql
import time
from data_config import *
from flask import request, make_response
import requests

# DB connect function
def db_con():
    db = pymysql.connect(
        host=host,
        user=user,
        password=password,
        charset='utf8mb4',
        database=database,
    )
    return db

#symbol load
def symbol_load():
    db = db_con()

    curs = db.cursor()
    sql = "select symbol from fchart group by symbol"
    curs.execute(sql)
    rows = curs.fetchall()

    options = list()
    for row in rows:
        if row[0] == 'BTCUPUSDT' or row[0] == 'BTCDOWNUSDT':
            continue
        options.append({'label':row[0], 'value':row[0]})

    db.close()
    return options

# tline load
def tline_load(is_ext):
    db = db_con()
    curs = db.cursor()
    if is_ext == 1:
        sql = "select symbol, rods, gname from %s where is_extension = 1" % (tline)
    else:
        sql = "select symbol, rods, gname from %s" % (tline)
    curs.execute(sql)
    rows = curs.fetchall()

    tline_opt = dict()
    for row in rows:
        if row[0]+row[1] not in tline_opt.keys():
            tline_opt[row[0]+row[1]] = list()
        tline_opt[row[0]+row[1]].append({'label':row[2], 'value':row[2]})
    db.close()
    return tline_opt

# graph data load function
def graph(sbl, rod):
    data = dict()
    data['Open'] = list()
    data['Close'] = list()
    data['High'] = list()
    data['Low'] = list()
    name_list = list()
    
    db = db_con()

    sql = "select open, close, high, low, dt from fchart where symbol = %s and rods = %s"
    curs = db.cursor()
    curs.execute(sql, (sbl, rod))
    rows = curs.fetchall()

    for row in rows:
        data['Open'].append(row[0])
        data['Close'].append(row[1])
        data['High'].append(row[2])
        data['Low'].append(row[3])
        name_list.append(row[4])
    db.close()
    return data, name_list

# trend line query function
def tline_query(sbl, rod):
    db = db_con()
    sql = "select * from %s where symbol = '%s' and rods = '%s'" % (tline, sbl, rod)
    curs = db.cursor()
    curs.execute(sql)
    row = curs.fetchall()
    db.close()
    return row

# trend line insert function
def tline_insert(sbl, rod, gname, fr_dt, fr_pc, to_dt, to_pc, is_extension):
    db = db_con()
    sql = """insert into %s(symbol, rods, gname, fr_dt, fr_pc, to_dt, to_pc, is_extension)
                    values (%s)""" % (tline, "%s, %s, %s, %s, %s, %s, %s, %s")
    curs = db.cursor()
    curs.execute(sql, (sbl, rod, gname, fr_dt, fr_pc, to_dt, to_pc, is_extension))
    db.commit()
    db.close()

# trend line delete function
def tline_delete(sbl, rod, gname):
    db = db_con()
    sql = """delete from %s where symbol ='%s' and rods = '%s' and gname = '%s'""" %(tline, sbl, rod, gname)
    curs = db.cursor()
    curs.execute(sql)
    db.commit()
    db.close()

# extension tline query function
def ext_tline_query(frdate, todate, frprice, toprice, rod, todt):
    if rod == 'min5':
        per = 300
        thd_time = int(time.strftime('%M'))//5
        to_datetime = datetime(year=int(todt[:4]), month=int(todt[4:6]), day=int(todt[6:8]), hour=int(todt[8:10]), minute=thd_time*5)
    elif rod == 'min15':
        per = 900
        thd_time = int(time.strftime('%M'))//15
        to_datetime = datetime(year=int(todt[:4]), month=int(todt[4:6]), day=int(todt[6:8]), hour=int(todt[8:10]), minute=thd_time*15)
    elif rod == 'min30':
        per = 1800
        thd_time = int(time.strftime('%M'))//30
        to_datetime = datetime(year=int(todt[:4]), month=int(todt[4:6]), day=int(todt[6:8]), hour=int(todt[8:10]), minute=thd_time*30)
    elif rod == 'hour1':
        per = 3600
        thd_time = int(time.strftime('%H'))
        to_datetime = datetime(year=int(todt[:4]), month=int(todt[4:6]), day=int(todt[6:8]), hour=thd_time)
    elif rod == 'hour4':
        per = 14400
        thd_time = int(time.strftime('%H'))/4
        to_datetime = datetime(year=int(todt[:4]), month=int(todt[4:6]), day=int(todt[6:8]), hour=thd_time*4)
    else:
        per = 86400
        thd_time = int(time.strftime('%d'))
        to_datetime = datetime(year=int(todt[:4]), month=int(todt[4:6]), day=thd_time,)
        
    xs = [datetime(year=int(frdate[:4]), month=int(frdate[4:6]), day=int(frdate[6:8]), hour=int(frdate[8:10]), minute=int(frdate[10:12])),
            datetime(year=int(todate[:4]), month=int(todate[4:6]), day=int(todate[6:8]), hour=int(todate[8:10]), minute=int(todate[10:12])),
            to_datetime]
    
    diff_min = ((xs[1]-xs[0]).total_seconds())/per
    diff_n_min = ((xs[2]-xs[1]).total_seconds())/per

    diff_price = round(toprice - frprice, 8)
    
    add_price = diff_price/diff_min
    
    ys = [frprice, toprice, round(toprice+(add_price*diff_n_min),8)]

    return xs, ys

# trading list
def trading_list_query(sbl, rod, is_complete):
    db = db_con()
    curs = db.cursor()
    if is_complete == 1:
        sql = """select * from %s where symbol = '%s' and rods = '%s' and is_complete > 0""" % (trading, sbl, rod)
    else:
        sql = """select * from %s where symbol = '%s' and rods = '%s' and is_complete < 1""" % (trading, sbl, rod)
    
    curs.execute(sql)
    rows = curs.fetchall()
    db.close()
    row_lists = list()

    for row in rows:
        row_lists.append({'gn':row[3], 'tm':row[4], 'bs':row[5], 'lm':row[6], 'qt':row[7], 'num':row[0]})

    return row_lists

# trading insert function
def trading_insert(sbl, rod, gname, timing, tdtype, pctype, qty):
    db = db_con()
    sql = """insert into %s(symbol, rods, gname, timing, tdtype, pctype, qty)
                    values (%s)"""  % (trading, "%s, %s, %s, %s, %s, %s, %s")
    curs = db.cursor()
    curs.execute(sql, (sbl, rod, gname, timing, tdtype, pctype, qty))
    db.commit()
    db.close()

# trading delete function
def trading_delete(open_rows, sbl, rod):
    # [{'gn': 'b', 'tm': 'breakthrough', 'bs': 'buy', 'lm': 'limits', 'qt': 1}, {'gn': 'c', 'tm': 'breakthrough', 'bs': 'buy', 'lm': 'limits', 'qt': 11}]
    row_lists = list()
    for row in open_rows:
        row_lists.append((row['gn'], row['tm'], row['bs'], row['lm'], row['num']))
    row_tuple = tuple(row_lists)
    
    db = db_con()
    curs = db.cursor()

    sql = """select gname, timing, tdtype, pctype, num from %s where symbol = '%s' and rods = '%s' and is_complete = 0""" % (trading, sbl, rod)
    curs.execute(sql)
    rows = curs.fetchall()
    
    diff_rows = tuple(set(rows) ^ set(row_tuple))

    if len(diff_rows) == 1:
        sql = """delete from %s where num = '%s'""" % (trading, diff_rows[0][4])
        curs.execute(sql)

        db.commit()
        db.close()





data, name_list = graph('BTCUSDT', 'min5')
fig = go.Figure(data=[go.Candlestick(x=name_list,
                        open=data['Open'], high=data['High'],
                        low=data['Low'], close=data['Close'],
                        name = 'BTCUSDT')])
fig.update_layout(height=700,)

options = symbol_load()
# tline_opt = tline_load(0)
# trd_inst_tline_opt = tline_load(1)

VALID_USERNAME_PASSWORD_PAIRS = {
    'hello': 'world'
}

# external_stylesheets = ['https://codepen.io/chriddyp/pen/bWLwgP.css']
external_stylesheets = ['https://codepen.io/chriddyp/pen/bWLwgP.css', 'bWLwgP.css']
app = Dash(__name__, external_stylesheets=external_stylesheets)

auth = BasicAuth(
    app,
    VALID_USERNAME_PASSWORD_PAIRS
)

app.layout = html.Div([
    # dcc.Graph(figure=fig),
    
    html.Div(
        [
            #html.Label('asdf', style={'display':'inline-block', 'vertical-align':'middle'}),
            html.Div([
            html.Div([
                dcc.Dropdown(
                    id='symbol',
                    options=options,
                    value='BTCUSDT',
                )
                ],
                style={
                    'display':'inline-block',
                    'width':'30%',
                    'padding-left':'5%',
                    'vertical-align':'middle'
                },
            ),
            html.Div([
            dcc.Dropdown(
                id='rods',
                options=[
                    {'label': '5 minutes', 'value': 'min5'},
                    {'label': '15 minutes', 'value': 'min15'},
                    {'label': '30 minutes', 'value': 'min30'},
                    {'label': '1 hours', 'value': 'hour1'},
                    {'label': '4 hours', 'value': 'hour4'},
                    {'label': '1 day', 'value': 'day1'},
                ],
                value='min5',
            ),
            ],
                style={
                    'display':'inline-block',
                    'width':'30%',
                    'vertical-align':'middle'
                }
            ),
            html.Div([
                html.Button('Submit', id='chg_button'),
            ],
                style={
                    'display':'inline-block',
                    'vertical-align':'middle'
                }
            ),
            ]),
            html.Div([dcc.Interval(id='graph-update', interval=1*280000)]),
            dcc.Loading(
            id="loading-1",
            children=[html.Div(id='figure-container', children=[dcc.Graph(figure=fig)])],
            type="circle"),
            # html.Button(id="input-1", children='Input triggers nested spinner')
        ], # style={'width': 400}
    ),

    # tline isnert
    html.Div([  
                html.Div([ 
                        dcc.Input(id='frdate', type='number', placeholder='from date 202005192000', style={'margin-right':'5px', 'width':'48%'}), 
                        dcc.Input(id='frprice', type='number', placeholder='0.00', style={'width':'49%'}), 
                    ],
                    style={
                        'padding':'10px 10px 0',
                    }
                ),
                html.Div([
                        dcc.Input(id='todate', type='number', placeholder='to date 202005202000', style={'margin-right':'5px', 'width':'48%'}), 
                        dcc.Input(id='toprice', type='number', placeholder='0.00', style={'width':'49%'}), 
                    ],
                    style={
                        'padding':'5px 10px 0',
                    }
                ),
                html.Div([
                        dcc.Input(id='gname', placeholder='graph_name', style={'margin-right':'5px', 'width':'48%'}),
                        dcc.Checklist(id='is_extension', options=[{'label': ' EXT', 'value':'1'},], style={'display':'inline-block', 'margin-right':'3%','width':'18.5%'}),
                        html.Button('Create', id='button', style={'width':'27.5%'}),          
                    ],
                    style={
                        'padding':'5px 10px 10px',
                    }
                ),
        ],
        style={
            'margin-left':'2%',
            'border':'1px solid gray',
            'border-radius':'5px',
            'float':'left',
            'width':'28%'
        }
    ),

    # trading insert
    html.Div([
        html.Div(
            id='trd_inst_tline_list', children=[
                                    dcc.Dropdown(
                                        id='trd_inst_tline_list_cmb',
                                        options=[],
                                        value='',
                                    )
                                ],
            style={
                'padding':'10px 5px 10px 10px',
                'width':'16%',
                'display':'inline-block'
            }
        ),
        html.Div(
            children=[
                dcc.Dropdown(
                    id='trd_inst_timing_cmb',
                    options=[{'label': '상향돌파', 'value':'breakthrough'},{'label': '하향돌파', 'value':'downwdbreak'},],
                    value='breakthrough',
                ),
            ],
            style={
                'padding':'10px 5px 10px 0',
                'display':'inline-block',
                'width':'16%',
            }
        ),
        html.Div(
            children=[
                dcc.Dropdown(
                    id='trd_inst_tdtype_cmb',
                    options=[{'label': '매수', 'value':'buy'},{'label': '매도', 'value':'sell'},],
                    value='buy',
                ),
            ],
            style={
                'padding':'10px 5px 10px 0',
                'display':'inline-block',
                'width':'16%',
            }
        ),
        html.Div(
            children=[
                dcc.Dropdown(
                    id='trd_inst_pctype_cmb',
                    options=[{'label': '시장가', 'value':'market'},{'label': '지정가', 'value':'limit'},],
                    value='market',
                ),
            ],
            style={
                'padding':'10px 5px 10px 0',
                'display':'inline-block',
                'width':'16%',
            }
        ),
        html.Div([ 
                dcc.Input(id='qty1', type='number', placeholder='Quantity of trading', style={'margin-right':'1%','width':'60%'}), 
                html.Button('Send', id='td_button', style={'width':'37.5%'}),  
            ],
            style={
                'padding':'10px 5px 10px',
                'display':'inline-block',
                'width':'31%',
                'float':'right',
            }
        ),

    ],
    style={
        'margin-right':'2%',
        'border':'1px solid gray',
        'border-radius':'5px',
        'float':'right',
        'width':'66%',
        'height':'55px'
    }
    ),

    # trading list
    html.Div([
            dcc.Tabs(id='trd_tabs', value='open_tab', children=[
                dcc.Tab(label='Open Orders', value='open_tab', style={'padding':'5px'}, selected_style={'padding':'5px'}),
                dcc.Tab(label='Order History', value='history_tab', style={'padding':'5px'}, selected_style={'padding':'5px'}),
            ],
            style={
                # 'height':'30px'
                'padding':'5px'
            }),
            html.Div(id='trd_detail_list', children=[
                # dash_table.DataTable(
                #     id='open-rows-table',
                #     columns=[{
                #         'name': 'Column {}'.format(i),
                #         'id': 'column-{}'.format(i),
                #         'deletable': True,
                #         'renamable': True
                #     } for i in range(1, 5)],
                #     data=[
                #         {'column-{}'.format(i): (j + (i-1)*5) for i in range(1, 5)}
                #         for j in range(5)
                #     ],
                #     editable=True,
                #     row_deletable=True
                # ),
            ])
        ],
        style={
            'margin-right':'2%',
            'margin-top':'10px',
            'margin-bottom':'15px',
            'border':'1px solid gray',
            'border-radius':'5px',
            'float':'right',
            'width':'66%',
            'height':'200px',
            'overflow':'scroll'
        }
    ),

    # tline delete
    html.Div([
        html.Div(
            id='tline_list', children=[
                                    dcc.Dropdown(
                                        id='tsbl',
                                        options=[],
                                        value='',
                                    )
                                ],
            style={
                'padding':'10px',
                'width':'186px',
                'display':'inline-block'
            }
        ),
        html.Div([
                html.Button('Delete', id='del_button', style={}),
            ],
            style={
                'float':'right',
                'margin-top':'8px',
                'margin-right':'10px'
            }
        )
        ],
        style={
            'margin-left':'2%',
            'margin-top':'10px',
            'border':'1px solid gray',
            'border-radius':'5px',
            'float':'left',
            'width':'28%',
            'height':'55px'
        }
    ),
    
    html.H2(id='show-output', children=''),
    html.Button('press to show username', id='logout_button')

    # html.Div(id='hidden_div',children='')
])


@app.callback(
    Output(component_id='show-output', component_property='children'),
    [Input(component_id='logout_button', component_property='n_clicks')]
)
def update_output_div(n_clicks):
    # request.authorization['username']
    # print(request.headers.get('Authorization'))
    # request.authorization.pop('username')
    s = requests.session()

    s.cookies.clear()
    print(s.cookies.keys())
    request.authorization.clear()
    print(request.authorization.keys())

# app.scripts.config.serve_locally = True

################
### callback ###
################

@app.callback(
    Output('figure-container', 'children'),
    [Input('button', 'n_clicks'), Input('chg_button', 'n_clicks'), Input('del_button', 'n_clicks'), Input('graph-update', 'n_intervals')],
    [State('is_extension','value'),State('gname', 'value'),
     State('frdate', 'value'), State('frprice', 'value'),
     State('todate', 'value'), State('toprice', 'value'),
     State('symbol', 'value'), State('rods','value'),
     State('tsbl', 'value')
    ]
)
def update_output(button, chg_button, del_button, n_intervals, is_extension, gname, v1, v2, v3, v4, sbl, rod, tsbl):
    ctx = callback_context
    if ctx.triggered[0]['value'] is None:
        button_id = 'No clicks yet'
    else:
        button_id = ctx.triggered[0]['prop_id'].split('.')[0]
    
    if button_id == 'button':
    
        if not v1 or not v2 or not v3 or not v4:
            pass
        else:
            data, name_list = graph(sbl, rod)
            fig2 = go.Figure(data=[go.Candlestick(x=name_list,
                                open=data['Open'], high=data['High'],
                                low=data['Low'], close=data['Close'],
                                )],
                             layout={'title':sbl+' / '+rod},
                            )
            fig2.update_layout(height=700,)
            # trand line insert
            try:
                ext = int(is_extension[0])
            except:
                ext = 0
            tline_insert(sbl, rod, gname, str(v1), v2, str(v3), v4, ext)
            # query trand line
            rows = tline_query(sbl, rod)

            for row in rows:
                frdate = row[3]
                todate = row[5]
                frprice = row[4]
                toprice = row[6]

                if row[7] == 1:
                    if 'min' in rod:
                        todt = time.strftime('%Y%m%d%H')
                    elif 'hour' in rod:
                        todt = time.strftime('%Y%m%d')
                    else:
                        todt = time.strftime('%Y%m')

                    xs, ys = ext_tline_query(frdate, todate, frprice, toprice, rod, todt)

                else:
                    xs = [datetime(year=int(frdate[:4]), month=int(frdate[4:6]), day=int(frdate[6:8]), hour=int(frdate[8:10]), minute=int(frdate[10:12])),
                            datetime(year=int(todate[:4]), month=int(todate[4:6]), day=int(todate[6:8]), hour=int(todate[8:10]), minute=int(todate[10:12]))]
                    ys = [row[4], row[6]]
                fig2.add_trace(go.Scatter(x= xs, y= ys, 
                                mode='lines+markers',
                                name=row[2]))
            return dcc.Graph(figure=fig2, id='figure-1')
    
    elif button_id == 'del_button':
        tline_delete(sbl, rod, tsbl)
        data, name_list = graph(sbl, rod)
        fig2 = go.Figure(data=[go.Candlestick(x=name_list,
                            open=data['Open'], high=data['High'],
                            low=data['Low'], close=data['Close'],
                            )],
                         layout={'title':sbl+' / '+rod},
                        )
        fig2.update_layout(height=700,)

        rows = tline_query(sbl, rod)
        for row in rows:
            frdate = row[3]
            todate = row[5]
            frprice = row[4]
            toprice = row[6]

            if row[7] == 1:
                if 'min' in rod:
                    todt = time.strftime('%Y%m%d%H')
                elif 'hour' in rod:
                    todt = time.strftime('%Y%m%d')
                else:
                    todt = time.strftime('%Y%m')

                xs, ys = ext_tline_query(frdate, todate, frprice, toprice, rod, todt)

            else:
                xs = [datetime(year=int(frdate[:4]), month=int(frdate[4:6]), day=int(frdate[6:8]), hour=int(frdate[8:10]), minute=int(frdate[10:12])),
                        datetime(year=int(todate[:4]), month=int(todate[4:6]), day=int(todate[6:8]), hour=int(todate[8:10]), minute=int(todate[10:12]))]
                ys = [row[4], row[6]]
            fig2.add_trace(go.Scatter(x= xs, y= ys,
                            mode='lines+markers',
                            name=row[2]))

        return dcc.Graph(figure=fig2, id='figure-1')

    else:
        data, name_list = graph(sbl, rod)
        fig2 = go.Figure(data=[go.Candlestick(x=name_list,
                            open=data['Open'], high=data['High'],
                            low=data['Low'], close=data['Close'],
                            )],
                         layout={'title':sbl+' / '+rod},
                        )
        fig2.update_layout(height=700,)

        rows = tline_query(sbl, rod)
        for row in rows:
            frdate = row[3]
            todate = row[5]
            frprice = row[4]
            toprice = row[6]

            if row[7] == 1:
                if 'min' in rod:
                    todt = time.strftime('%Y%m%d%H')
                elif 'hour' in rod:
                    todt = time.strftime('%Y%m%d')
                else:
                    todt = time.strftime('%Y%m')

                xs, ys = ext_tline_query(frdate, todate, frprice, toprice, rod, todt)

            else:
                xs = [datetime(year=int(frdate[:4]), month=int(frdate[4:6]), day=int(frdate[6:8]), hour=int(frdate[8:10]), minute=int(frdate[10:12])),
                        datetime(year=int(todate[:4]), month=int(todate[4:6]), day=int(todate[6:8]), hour=int(todate[8:10]), minute=int(todate[10:12]))]
                ys = [row[4], row[6]]
            fig2.add_trace(go.Scatter(x= xs, y= ys,
                            mode='lines+markers',
                            name=row[2]))

        return dcc.Graph(figure=fig2, id='figure-1')


@app.callback(
    Output('tline_list', 'children'),
    [Input('button', 'n_clicks'), Input('chg_button', 'n_clicks'), Input('del_button', 'n_clicks')],
    [State('symbol', 'value'), State('rods','value')]
)
def update_tline_list(button, chg_button, del_button, sbl, rod):
    time.sleep(1)
    tline_opt = tline_load(0)
    try:        
        tline_list = [
            dcc.Dropdown(
                id='tsbl',
                options=tline_opt[sbl+rod],
                value='',
            )
        ]
    except:
        tline_list = [
            dcc.Dropdown(
                id='tsbl',
                options=[],
                value='',
            )
        ]
    return html.Div(tline_list)


@app.callback(
    Output('trd_inst_tline_list', 'children'),
    [Input('button', 'n_clicks'), Input('chg_button', 'n_clicks'), Input('del_button', 'n_clicks')],
    [State('symbol', 'value'), State('rods','value')]
)
def update_trd_inst_tline_list(button, chg_button, del_button, sbl, rod):
    time.sleep(1)
    trd_inst_tline_opt = tline_load(1)
    try:        
        trd_inst_tline_list = [
            dcc.Dropdown(
                id='trd_inst_tline_list_cmb',
                options=trd_inst_tline_opt[sbl+rod],
                value='',
            )
        ]
    except:
        trd_inst_tline_list = [
            dcc.Dropdown(
                id='trd_inst_tline_list_cmb',
                options=[],
                value='',
            )
        ]
    return html.Div(trd_inst_tline_list)


@app.callback(Output('trd_detail_list', 'children'),
            [Input('trd_tabs', 'value'), Input('chg_button', 'n_clicks'), Input('td_button', 'n_clicks')],
            [
                State('symbol', 'value'), State('rods','value'),
                State('trd_inst_tline_list_cmb', 'value'), State('trd_inst_timing_cmb','value'), State('trd_inst_tdtype_cmb', 'value'), 
                State('trd_inst_pctype_cmb','value'), State('qty1','value'),
            ])
def render_content(tab, chg_btn, btn, sbl, rod, gname, timing, tdtype, pctype, qty):
    ctx = callback_context
    if ctx.triggered[0]['value'] is None:
        button_id = 'No clicks yet'
    else:
        button_id = ctx.triggered[0]['prop_id'].split('.')[0]
    # if td_button (insert)
    if button_id == 'td_button':
        trading_insert(sbl, rod, gname, timing, tdtype, pctype, qty)
        time.sleep(1)

    if tab == 'open_tab':
        rows = trading_list_query(sbl, rod, 0)
        return DataTable(
                    id='open-rows-table',
                    columns=[
                        {'name': 'Graph Name', 'id': 'gn',},
                        {'name': 'Timing', 'id': 'tm',},
                        {'name': 'Buy/Sell', 'id': 'bs',},
                        {'name': 'Limit/Market', 'id': 'lm',},
                        {'name': 'Quantity', 'id': 'qt',},
                        {'name': 'orderId', 'id': 'num',},
                    ],
                    data=rows,
                    editable=False,
                    row_deletable=True
                ),
    elif tab == 'history_tab':
        rows = trading_list_query(sbl, rod, 1)
        return DataTable(
                    id='history-rows-table',
                    columns=[
                        {'name': 'Graph Name', 'id': 'gn',},
                        {'name': 'Timing', 'id': 'tm',},
                        {'name': 'Buy/Sell', 'id': 'bs',},
                        {'name': 'Limit/Market', 'id': 'lm',},
                        {'name': 'Quantity', 'id': 'qt',},
                        {'name': 'orderId', 'id': 'num',},
                    ],
                    data=rows,
                    editable=False,
                    row_deletable=False
                ),


@app.callback(
    Output('hidden_div', 'children'),
    [Input('open-rows-table', 'data'),],
    [State('symbol', 'value'), State('rods','value'),]
)
def display_output(open_rows, sbl, rod):
    trading_delete(open_rows, sbl, rod)
    return open_rows

# app.run_server(debug=True, use_reloader=False)
if __name__ == '__main__':
    # For Development only, otherwise use gunicorn or uwsgi to launch, e.g.
    # gunicorn -b 0.0.0.0:8050 index:app.server
    app.run_server(
        port=port,
        host='0.0.0.0',
        debug=False
    )
