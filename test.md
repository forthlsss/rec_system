## 基于flask框架实现租房网站
### 操作步骤
1.创建项目\
2.配置数据库\
3.定义模型类\
4.定义视图并配置URL\
5.定义模板\
### 配置文件 
1.配置文件：\

设置：config.py\

主文件：ihome.py\

图片上传：qinniuyun_sdk.py\

短信发送：ytx_send.py\

管理文件：manager.py
\
状态文件：status_code.py\

模型设置：models.py

\
2.视图：\
user_views.py

#### 配置文件


1.配置文件：
设置：config.py、数据库、七牛云配置\
```

# coding=utf-8
import hashlib
 
import redis
class Config:
    DEBUG=False
    SQLALCHEMY_DATABASE_URI='mysql://root:mysql@127.0.0.1:3306/ihome_bj14'
    SQLALCHEMY_TRACK_MODIFICATIONS = True
#PIL pillow
    #redis配置
    REDIS_HOST = "127.0.0.1"
    REDIS_PORT = 6379
 
    #session配置
    SECRET_KEY = "iHome"
    #将session存储到redis中
    SESSION_TYPE = "redis"
    SESSION_USE_SIGNER = True
    SESSION_REDIS = redis.StrictRedis(host=REDIS_HOST, port=REDIS_PORT)
    PERMANENT_SESSION_LIFETIME = 60*60*24*14#秒
 
    # 七牛云的访问服务器
    QINIU_URL = 'http://ozyr1iz9u.bkt.clouddn.com/'
 
    # # 权限信息
    # TOKEN = hashlib.md5('ihome_bj14').hexdigest()
 
 
class DevelopConfig(Config):
    DEBUG = True
class ProductConfig(Config):
    pass
```
主文件：ihome.py
```

#coding=utf-8
 
#1.创建app对象
from manager import create_app
from config import DevelopConfig
app=create_app(DevelopConfig)
 
#2.初始化数据库
from models import db
db.init_app(app)
 
#3.创建管理对象
from flask_script import Manager
manager=Manager(app)
 
#4.添加迁移命令
from flask_migrate import Migrate,MigrateCommand
Migrate(app,db)
manager.add_command('db',MigrateCommand)
 
#5.注册蓝图
from html_views import html_blueprint
app.register_blueprint(html_blueprint)
 
from api_v1.user_views import user_blueprint
app.register_blueprint(user_blueprint,url_prefix='/api/v1/user')
 
from api_v1.house_views import house_blueprint
app.register_blueprint(house_blueprint,url_prefix='/api/v1/house')
 
from api_v1.order_views import order_blueprint
app.register_blueprint(order_blueprint,url_predix='/api/v1/order')
 
# # 钩子，是否包含token
# from flask import request,jsonify,current_app
# from status_code import *
# @app.before_request
# def check_token():
#     if request.path.startswith('/api'):
#         if 'token' not in request.args or request.args.get('token')!=current_app.config['TOKEN']:
#             return jsonify(code=RET.REQERR)
 
if __name__ == '__main__':
    manager.run()
 
```
图片上传
```

#coding=utf-8
 
from qiniu import Auth,put_data
import logging
from flask import jsonify
from status_code import RET
 
def put_qiniu(f1):
    access_key = '6vvGPJ08BsHRjN7RRGVU3sEd38502x1TS0_i0x7s'
    secret_key = 'wzgWlHpflABi9DGn9TIb8IBRwqsZTwNJ--KZhosy'
    # 空间名称
    bucket_name = 'itpython'
 
    try:
        #构建鉴权对象
        q = Auth(access_key,secret_key)
        #生成上传Token
        token = q.upload_token(bucket_name)
        #上传文件数据，ret是字典，键是hash/key,值是新文件名，info是response对象
        ret,info = put_data(token,None,f1.read())
        return ret.get('key')
    except:
        logging.ERROR(u'访问七牛云出错')
        return jsonify(code=RET.SERVERERR)
```
短信发送：ytx_send.py
```

# coding=utf-8
from CCPRestSDK import REST
# import ConfigParser
 
# 主帐号
accountSid = '8a216da85fe1c856015fe83104160cc';
 
# 主帐号Token
accountToken = '5a1dc45a953a41cebb82b5654662568';
 
# 应用Id
appId = '8a216da85fe1c856015fe8310471033';
 
# 请求地址，格式如下，不需要写http://
serverIP = 'app.cloopen.com';
 
# 请求端口
serverPort = '8883';
 
# REST版本号
softVersion = '2013-12-26';
 
# 发送模板短信
# @param to 手机号码
# @param datas 内容数据 格式为数组 例如：{'12','34'}，如不需替换请填 ''
# @param $tempId 模板Id
'''
to: 短信接收手机号码集合,用英文逗号分开,如 '13810001000,13810011001',最多一次发送200个。
datas：内容数据，需定义成数组方式，如模板中有两个参数，定义方式为array['Marry','Alon']。
templateId: 模板Id,如使用测试模板，模板id为"1"，如使用自己创建的模板，则使用自己创建的短信模板id即可。
'''
 
def sendTemplateSMS(to, datas, tempId):
 
    # 初始化REST SDK
    rest = REST(serverIP, serverPort, softVersion)
    rest.setAccount(accountSid, accountToken)
    rest.setAppId(appId)
 
    result = rest.sendTemplateSMS(to, datas, tempId)
    return result.get('statusCode')
```
管理文件：manager.py
```

# coding=utf-8
from flask import Flask
from redis import StrictRedis
from werkzeug.routing import BaseConverter
import os
from flask_session import Session
 
BASE_DIR=os.path.dirname(os.path.abspath(__file__))
 
class HTMLConverter(BaseConverter):
    regex = '.*'
 
 
def create_app(config):
    app=Flask(__name__)
    app.config.from_object(config)
    app.url_map.converters['html']=HTMLConverter
 
    #session初始化，将session存储到redis中
    Session(app)
 
    # 创建redis存储对象
    redis_store = StrictRedis(host=config.REDIS_HOST, port=config.REDIS_PORT)
    app.redis = redis_store
 
    # 日志处理
    import logging
    from logging.handlers import RotatingFileHandler
    logging.basicConfig(level=logging.DEBUG)
    file_log_handler = RotatingFileHandler(os.path.join(BASE_DIR, "logs/ihome.log"), maxBytes=1024 * 1024 * 100,backupCount=10)
    formatter = logging.Formatter('%(levelname)s %(filename)s:%(lineno)d %(message)s')
    file_log_handler.setFormatter(formatter)
    logging.getLogger().addHandler(file_log_handler)
 
    return app
```
状态文件：status_code.py
```

# coding=utf-8
class RET:
    OK = "0"
    DBERR = "4001"
    NODATA = "4002"
    DATAEXIST = "4003"
    DATAERR = "4004"
    SESSIONERR = "4101"
    LOGINERR = "4102"
    PARAMERR = "4103"
    USERERR = "4104"
    ROLEERR = "4105"
    PWDERR = "4106"
    REQERR = "4201"
    IPERR = "4202"
    THIRDERR = "4301"
    IOERR = "4302"
    SERVERERR = "4500"
    UNKOWNERR = "4501"
 
ret_map = {
    RET.OK: u"成功",
    RET.DBERR: u"数据库查询错误",
    RET.NODATA: u"无数据",
    RET.DATAEXIST: u"数据已存在",
    RET.DATAERR: u"数据错误",
    RET.SESSIONERR: u"用户未登录",
    RET.LOGINERR: u"用户登录失败",
    RET.PARAMERR: u"参数错误",
    RET.USERERR: u"用户不存在或未激活",
    RET.ROLEERR: u"用户身份错误",
    RET.PWDERR: u"密码错误",
    RET.REQERR: u"非法请求或请求次数受限",
    RET.IPERR: u"IP受限",
    RET.THIRDERR: u"第三方系统错误",
    RET.IOERR: u"文件读写错误",
    RET.SERVERERR: u"内部错误",
    RET.UNKOWNERR: u"未知错误",
    }
```
模型设置：models.py主要根据网页来寻找设置对象
```

# coding=utf-8
from flask_sqlalchemy import SQLAlchemy
from datetime import datetime
from werkzeug.security import generate_password_hash,check_password_hash
from flask import current_app
 
db=SQLAlchemy()
 
class BaseModel(object):
    create_time=db.Column(db.DATETIME,default=datetime.now())
    update_time=db.Column(db.DATETIME,default=datetime.now(),onupdate=datetime.now())
 
    def add_update(self):
        db.session.add(self)
        db.session.commit()
    def delete(self):
        db.session.delete(self)
        db.session.commit()
 
class User(BaseModel,db.Model):
    __tablename__='ihome_user'
    id=db.Column(db.INTEGER,primary_key=True)
    phone=db.Column(db.String(11),unique=True)
    pwd_hash=db.Column(db.String(200))
    name=db.Column(db.String(30),unique=True)
    avatar=db.Column(db.String(100))
    id_name=db.Column(db.String(30))
    id_card=db.Column(db.String(18),unique=True)
 
    houses=db.relationship('House',backref='user')
    orders=db.relationship('Order',backref='user')
 
    #读
    @property
    def password(self):
        return ''
    #写
    @password.setter
    def password(self,pwd):
        self.pwd_hash=generate_password_hash(pwd)
 
    #对比
    def check_pwd(self,pwd):
        return check_password_hash(self.pwd_hash,pwd)
 
    def to_basic_dict(self):
        return {
            'id':self.id,
            'avatar':current_app.config['QINIU_URL']+self.avatar if self.avatar else '',
            'name':self.name,
            'phone':self.phone
        }
 
    def to_auth_dict(self):
        return {
            'id_name':self.id_name,
            'id_card':self.id_card
        }
 
ihome_house_facility = db.Table(
    "ihome_house_facility",
    db.Column("house_id", db.Integer, db.ForeignKey("ihome_house.id"), primary_key=True),  # 房屋编号
    db.Column("facility_id", db.Integer, db.ForeignKey("ihome_facility.id"), primary_key=True)  # 设施编号
)
 
class House(BaseModel, db.Model):
    """房屋信息"""
 
    __tablename__ = "ihome_house"
 
    id = db.Column(db.Integer, primary_key=True)  # 房屋编号
    # 房屋主人的用户编号
    user_id = db.Column(db.Integer, db.ForeignKey("ihome_user.id"), nullable=False)
    # 归属地的区域编号
    area_id = db.Column(db.Integer, db.ForeignKey("ihome_area.id"), nullable=False)
    title = db.Column(db.String(64), nullable=False)  # 标题
    price = db.Column(db.Integer, default=0)  # 单价，单位：分
    address = db.Column(db.String(512), default="")  # 地址
    room_count = db.Column(db.Integer, default=1)  # 房间数目
    acreage = db.Column(db.Integer, default=0)  # 房屋面积
    unit = db.Column(db.String(32), default="")  # 房屋单元， 如几室几厅
    capacity = db.Column(db.Integer, default=1)  # 房屋容纳的人数
    beds = db.Column(db.String(64), default="")  # 房屋床铺的配置
    deposit = db.Column(db.Integer, default=0)  # 房屋押金
    min_days = db.Column(db.Integer, default=1)  # 最少入住天数
    max_days = db.Column(db.Integer, default=0)  # 最多入住天数，0表示不限制
    order_count = db.Column(db.Integer, default=0)  # 预订完成的该房屋的订单数
    index_image_url = db.Column(db.String(256), default="")  # 房屋主图片的路径
 
    # 房屋的设施
    facilities = db.relationship("Facility", secondary=ihome_house_facility)
    images = db.relationship("HouseImage")  # 房屋的图片
    orders=db.relationship('Order',backref='house')
 
    def to_dict(self):
        return {
            'id':self.id,
            'title':self.title,
            'image':current_app.config['QINIU_URL']+self.index_image_url if self.index_image_url else '',
            'area':self.area.name,
            'price':self.price,
            'create_time':self.create_time.strftime('%Y-%m-%d %H:%M:%S'),
            'avatar':current_app.config['QINIU_URL']+self.user.avatar if self.user.avatar else '',
            'room':self.room_count,
            'order_count':self.order_count,
            'address':self.address
        }
 
    def to_full_dict(self):
        return {
            'id':self.id,
            'user_avatar':current_app.config['QINIU_URL']+self.user.avatar if self.user.avatar else '',
            'user_name':self.user.name,
            'price':self.price,
            'address':self.area.name+self.address,
            'room_count':self.room_count,
            'acreage':self.acreage,
            'unit':self.unit,
            'capacity':self.capacity,
            'beds':self.beds,
            'deposit':self.deposit,
            'min_days':self.min_days,
            'max_days':self.max_days,
            'order_count':self.order_count,
            'images':[current_app.config['QINIU_URL']+image.url for image in self.images],
            'facilities':[facility.to_dict() for facility in self.facilities],
        }
 
class HouseImage(BaseModel, db.Model):
    """房屋图片"""
 
    __tablename__ = "ihome_house_image"
 
    id = db.Column(db.Integer, primary_key=True)
    # 房屋编号
    house_id = db.Column(db.Integer, db.ForeignKey("ihome_house.id"), nullable=False)
    url = db.Column(db.String(256), nullable=False)  # 图片的路径
 
class Facility(BaseModel, db.Model):
    """设施信息"""
 
    __tablename__ = "ihome_facility"
 
    id = db.Column(db.Integer, primary_key=True)  # 设施编号
    name = db.Column(db.String(32), nullable=False)  # 设施名字
    css = db.Column(db.String(30), nullable=False)  # 设施展示的图标
 
    def to_dict(self):
        return {
            'id':self.id,
            'name':self.name,
            'css':self.css
        }
    def to_house_dict(self):
        return {'id':self.id}
 
class Area(BaseModel, db.Model):
    """城区"""
 
    __tablename__ = "ihome_area"
 
    id = db.Column(db.Integer, primary_key=True)  # 区域编号
    name = db.Column(db.String(32), nullable=False)  # 区域名字
    houses = db.relationship("House", backref="area")  # 区域的房屋
 
    def to_dict(self):
        return {
            'id':self.id,
            'name':self.name
        }
 
class Order(BaseModel,db.Model):
    __tablename__ = "ihome_order"
 
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey("ihome_user.id"), nullable=False)
    house_id = db.Column(db.Integer, db.ForeignKey("ihome_house.id"), nullable=False)
    begin_date = db.Column(db.DateTime, nullable=False)
    end_date = db.Column(db.DateTime, nullable=False)
    days = db.Column(db.Integer, nullable=False)
    house_price = db.Column(db.Integer, nullable=False)
    amount = db.Column(db.Integer, nullable=False)
    status = db.Column(
        db.Enum(
            "WAIT_ACCEPT",  # 待接单,
            "WAIT_PAYMENT",  # 待支付
            "PAID",  # 已支付
            "WAIT_COMMENT",  # 待评价
            "COMPLETE",  # 已完成
            "CANCELED",  # 已取消
            "REJECTED"  # 已拒单
        ),
        default="WAIT_ACCEPT", index=True)
    comment = db.Column(db.Text)
 
    def to_dict(self):
        return {
            'order_id':self.id,
            'house_title':self.house.title,
            'image':current_app.config['QINIU_URL']+self.house.index_image_url if self.house.index_image_url else '',
            'create_date':self.create_time.strftime('%Y-%m-%d'),
            'begin_date':self.begin_date.strftime('%Y-%m-%d'),
            'end_date':self.end_date.strftime('%Y-%m-%d'),
            'amount':self.amount,
            'days':self.days,
            'status':self.status,
            'comment':self.comment
        }
```
模型设定好之后，执行数据库迁移\
2.视图：user_views.py
```

#coding=utf-8
import random
import re
 
import logging
from flask import Blueprint, jsonify
from flask import current_app
from flask import make_response
from flask import request
from flask import session
from qiniu_sdk import put_qiniu
 
from captcha.captcha import captcha
from models import User
from status_code import RET, ret_map
from ytx_sdk.ytx_send import sendTemplateSMS
from my_decorators import is_login
 
user_blueprint=Blueprint('user',__name__)
 
 
#验证码
@user_blueprint.route('/yzm')
def yzm():
    name,text,image=captcha.generate_captcha()
    session['image_yzm']=text
    response=make_response(image)
    response.headers['Content-Type']='image/jpeg'
    return response
 
#短信发送设置
@user_blueprint.route('/send_sms')
def send_sms():
    #接收请求的数据
    dict=request.args
    mobile=dict.get('mobile')
    imageCode=dict.get('imageCode')
 
    #验证参数是否存在
    if not all([mobile,imageCode]):
        return jsonify(code=RET.PARAMERR,msg=ret_map[RET.PARAMERR])
    #验证手机号格式是否正确
    if not re.match(r'^1[345789]\d{9}$',mobile):
        return jsonify(code=RET.PARAMERR,msg=u'手机号格式不正确')
    #验证手机号是否存在
    if User.query.filter_by(phone=mobile).count():
        return jsonify(code=RET.PARAMERR,msg=u'手机号已存在')
    #验证图片验证码
    if imageCode!=session['image_yzm']:
        return jsonify(code=RET.PARAMERR,msg=u'图片验证码错误')
    #通过云通讯函数进行短信发送
    sms_code=random.randint(1000,9999)
    session['sms_yzm']=sms_code
    print(sms_code)
    result='000000'
    # result=sendTemplateSMS(mobile,[sms_code,'5'],1)
    #根据云通讯返回的结果进行相应
    if result=='000000':
        return jsonify(code=RET.OK,msg=ret_map[RET.OK])
    else:
        return jsonify(code=RET.UNKOWNERR,msg=u'短信发送失败')
 
#用户注册
@user_blueprint.route('/',methods=["POST"])
def user_register():
    #接收参数
    dict=request.form
    mobile=dict.get('mobile')
    imagecode=dict.get('imagecode')
    phonecode=dict.get('phonecode')
    password=dict.get('password')
    password2=dict.get('password2')
    #验证参数是否存在
    if not all([mobile,imagecode,phonecode,password,password2]):
        return jsonify(code=RET.PARAMERR,msg=ret_map[RET.PARAMERR])
    #验证图片验证码
    if imagecode!=session['image_yzm']:
        return jsonify(code=RET.PARAMERR,msg=u'图片验证码错误')
    #验证短信验证码
    if int(phonecode)!=session['sms_yzm']:
        return jsonify(code=RET.PARAMERR,msg=u'短信验证码错误')
    #验证手机号格式是否正确
    if not re.match(r'^1[345789]\d{9}$',mobile):
        return jsonify(code=RET.PARAMERR,msg=u'手机格式错误')
    #验证手机号是否存在
    if User.query.filter_by(phone=mobile).count():
        return jsonify(code=RET.PARAMERR,msg=u'手机号码存在')
    #保存用户对象
    user=User()
    user.phone=mobile
    user.name=mobile
    user.password=password
 
    try:
        user.add_update()
        return jsonify(code=RET.OK,msg=ret_map[RET.OK])
    except:
        logging.ERROR(u'用户注册更新数据库失败，手机号：%s,密码：%s'%(mobile,password))
        return jsonify(code=RET.DBERR,msg=ret_map[RET.DBERR])
 
@is_login
@user_blueprint.route('/',methods=['GET'])
def user_my():
    #获取当前登录的用户
    user_id=session['user_id']
    #查询当前用户的头像/用户名/手机号/并返回
    user=User.query.get(user_id)
    return jsonify(user=user.to_basic_dict())
 
@is_login
#个人信息获取用户名
@user_blueprint.route('/auth',methods=['GET'])
def user_auth():
    #获取当前登录用户的编号
    user_id=session['user_id']
    #根据编号查询当前用户
    user=User.query.get(user_id)
    #返回用户的真实姓名，身份证号
    return jsonify(user.to_auth_dict())
 
@is_login
#上传头像
@user_blueprint.route('/',methods=['PUT'])
def user_profile():
    dict=request.form
    if 'avatar1' in dict:
        try:
            #获取头像文件
            f1=request.files['avatar']
            # print(f1)
            # print(type(f1))
            # from werkzeug.datastructures import FileStorage
            #mime-type:国际规范，表示文件的类型，如text/html,text/xml,image/png,image/jpeg..
            if not re.match('image/.*',f1.mimetype):
                return jsonify(code=RET.PARAMERR)
        except:
            return jsonify(code=RET.PARAMERR)
        # 上传到七牛云
        # access_key = 'H999S3riCJGPiJOity1GsyWufw3IyoMB6goojo5e'
        # secret_key = 'uOZfRdFtljIw7b8jr6iTG-cC6wY_-N19466PXUAb'
        # # 空间名称
        # bucket_name = 'itcast20171104'
        url=put_qiniu(f1)
        #如果未出错
        #保存用户的头像信息
        try:
            user=User.query.get(session['user_id'])
            user.avatar=url
            user.add_update()
        except:
            logging.ERROR(u'数据库访问失败')
            return jsonify(code=RET.DBERR)
        # 则返回图片信息
        return jsonify(code=RET.OK,
                       url=current_app.config['QINIU_URL'] +  url)
    elif 'name' in dict:
        #修改用户名
        name=dict.get('name')
        #判断用户名是否存在
        if User.query.filter_by(name=name).count():
            return jsonify(code=RET.DATAEXIST)
        else:
            user=User.query.get(session['user_id'])
            user.name=name
            user.add_update()
            return jsonify(code=RET.OK)
    else:
        return jsonify(code=RET.PARAMERR,msg=ret_map[RET.PARAMERR])
 
@is_login
#实名认证
@user_blueprint.route('/auth',methods=['PUT'])
def user_auth_set():
    #接受参数
    dict=request.form
    id_name=dict.get('id_name')
    id_card=dict.get('id_card')
    #验证参数合法性
    if not all([id_name,id_card]):
        return jsonify(code=RET.PARAMERR,msg=ret_map[RET.SESSIONERR])
    #验证身份合法性
    if not re.match(r'^[1-9]\d{5}(19|20)\d{2}((0[1-9])|(10|11|12))(([0-2][1-9])|10|20|30|31)\d{3}[0-9Xx]$', id_card):
        return jsonify(code=RET.PARAMERR, msg=ret_map[RET.PARAMERR])
    #判断身份证号是否存在
    #修改数据对象
    try:
        user=User.query.get(session['user_id'])
    except:
        logging.ERROR(u'查询用户失败')
        return jsonify(code=RET.DBERR)
 
    try:
        user.id_card=id_card
        user.id_name=id_name
        user.add_update()
    except:
        logging.ERROR(u'修改用户姓名/身份证号失败')
        return jsonify(code=RET.DBERR)
 
    #返回数据
    return jsonify(code=RET.OK)
 
#用户注册
@user_blueprint.route('/session',methods=['POST'])
def user_login():
    #接收参数
    dict=request.form
    mobile=dict.get('mobile')
    password=dict.get('password')
    #验证非空
    if not all([mobile,password]):
        return jsonify(code=RET.PARAMERR,msg=ret_map[RET.PARAMERR])
    #验证手机号格式是否正确
    if not re.match(r'^1[345789]\d{9}$',mobile):
        return jsonify(code=RET.PARAMERR,msg=u'手机号格式错误')
    #数据处理
    try:
        user=User.query.filter_by(phone=mobile).first()
    except:
        logging.ERROR('用户登录--数据库出错')
        return jsonify(code=RET.DBERR,msg=ret_map[RET.DBERR])
    #判断手机号是否存在
    if user:
        #判断密码是否正确
        if user.check_pwd(password):
            session['user_id']=user.id
            return jsonify(code=RET.OK,msg=u'ok')
        else:
            return jsonify(code=RET.PARAMERR,msg=u'密码不正确')
    else:
        return jsonify(code=RET.PARAMERR,mag=u'手机号不存在')
 
#用户登录
@user_blueprint.route('/session',methods=['GET'])
def user_is_login():
    if 'user_id' in session:
        user=User.query.filter_by(id=session['user_id']).first()
        return jsonify(code=RET.OK,name=user.name)
    else:
        return jsonify(code=RET.DATAERR)
 
#用户退出
@user_blueprint.route('/session',methods=['DELETE'])
def user_logout():
    session.clear()
    return jsonify(code=RET.OK)
```
