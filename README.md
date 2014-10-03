AWS VPC入門
===
# 目的
AWS Virtual Private Cloudを使えるようにする。

# 前提
| ソフトウェア     | バージョン    | 備考         |
|:---------------|:-------------|:------------|
| AWS CloudFormation  |API Version 2010-05-15        |             |
| Amazon Virtual Private Cloud  |API Version 2014-02-01       |             |
|           　　　|        |             |

# 構成
+ [セットアップ](#0)
+ [ネットワークを構築する](#1)
+ [サーバーを構築する](#2)
+ [Webサーバーをセットアップする](#3)
+ [プライベートサブネットを構築する](#4)
+ [NATサーバーを構築する](#5)
+ [DBを用いたブログシステムの構築](#6)

# 詳細
## <a name="0">セットアップ</a>
EC2インスタンスにログインするためのキーペアを作成しておく。

![000](https://farm4.staticflickr.com/3931/15424184472_dace4a410c.jpg)

## <a name="1">ネットワークを構築する</a>

### AWS CloudFormationを使って構築

![001](https://farm4.staticflickr.com/3935/15237903678_8c755f3b5c.jpg)

![002](https://farm4.staticflickr.com/3934/15424156562_e4f633bec6.jpg)

![003](https://farm6.staticflickr.com/5599/15424156492_b11500a235.jpg)

![004](https://farm6.staticflickr.com/5597/15424496355_1cdfd73448.jpg)

![005](https://farm3.staticflickr.com/2944/15424156452_cca9c30a8e.jpg)

### ネットワーク図

![step01](https://farm6.staticflickr.com/5598/15421319621_ef766a9599.jpg)

### VPC領域
```json
"vpc2bcf394e": {
  "Type": "AWS::EC2::VPC",
  "Properties": {
    "CidrBlock": "10.0.0.0/16",
    "InstanceTenancy": "default",
    "EnableDnsSupport": "true",
    "EnableDnsHostnames": "true",
    "Tags": [
      {
        "Key": "Name",
        "Value": "VPC領域"
      }
    ]
  }
```

### インターネットゲートウェイ

```json
"igwdf809abd": {
  "Type": "AWS::EC2::InternetGateway",
  "Properties": {
  }
},
```

### パブリックサブネット

```json
"subnet4fb04638": {
  "Type": "AWS::EC2::Subnet",
  "Properties": {
    "CidrBlock": "10.0.1.0/24",
    "AvailabilityZone": "ap-northeast-1a",
    "VpcId": {
      "Ref": "vpc2bcf394e"
    },
    "Tags": [
      {
        "Key": "Name",
        "Value": "パブリックサブネット"
      }
    ]
  }
},
```

### パブリックルートテーブル

```json
"rtb3932c15c": {
  "Type": "AWS::EC2::RouteTable",
  "Properties": {
    "VpcId": {
      "Ref": "vpc2bcf394e"
    },
    "Tags": [
      {
        "Key": "Name",
        "Value": "パブリックルートテーブル"
      }
    ]
  }
},
"gw2": {
  "Type": "AWS::EC2::VPCGatewayAttachment",
  "Properties": {
    "VpcId": {
      "Ref": "vpc2bcf394e"
    },
    "InternetGatewayId": {
      "Ref": "igwdf809abd"
    }
  }
},
"subnetroute1": {
  "Type": "AWS::EC2::SubnetRouteTableAssociation",
  "Properties": {
    "RouteTableId": {
      "Ref": "rtb3932c15c"
    },
    "SubnetId": {
      "Ref": "subnet4fb04638"
    }
  }
},
"route1": {
  "Type": "AWS::EC2::Route",
  "Properties": {
    "DestinationCidrBlock": "0.0.0.0/0",
    "RouteTableId": {
      "Ref": "rtb3932c15c"
    },
    "GatewayId": {
      "Ref": "igwdf809abd"
    }
  },
  "DependsOn": "gw2"
},
```

パブリックルートテーブルの中身

| ディスティネーション（宛先）     | ターゲット    |
|:---------------|:-------------|
| 10.0.0.0/16    |local         |
|  0.0.0.0/0　　　 |インターネットゲートウェイ     |


## <a name="2">サーバーを構築する</a>

### AWS CloudFormation更新

![009](https://farm3.staticflickr.com/2948/15401841396_89fb4c28ab.jpg)

![010](https://farm3.staticflickr.com/2945/15424891045_c67d29c880.jpg)

### ネットワーク図

![Step02](https://farm3.staticflickr.com/2949/15238346547_ee71cbd547.jpg)

### Webサーバーインスタンス

```json
"instanceif16929e8": {
  "Type": "AWS::EC2::Instance",
  "Properties": {
    "DisableApiTermination": "FALSE",
    "ImageId": "ami-35072834",
    "InstanceType": "t2.micro",
    "KeyName": "my-key",
    "Monitoring": "false",
    "Tags": [
      {
        "Key": "Name",
        "Value": "Webサーバー"
      }
    ],
    "NetworkInterfaces": [
      {
        "DeleteOnTermination": "true",
        "Description": "Primary network interface",
        "DeviceIndex": 0,
        "SubnetId": {
          "Ref": "subnet2eb14759"
        },
        "PrivateIpAddresses": [
          {
            "PrivateIpAddress": "10.0.1.10",
            "Primary": "true"
          }
        ],
        "GroupSet": [
          {
            "Ref": "sgWEBSG"
          }
        ],
        "AssociatePublicIpAddress": "true"
      }
    ]
  }
},
```

### セキュリティグループ

```json
"sgWEBSG": {
  "Type": "AWS::EC2::SecurityGroup",
  "Properties": {
    "GroupDescription": "WEB-SG",
    "VpcId": {
      "Ref": "vpc9bcf39fe"
    },
    "SecurityGroupIngress": [
      {
        "IpProtocol": "tcp",
        "FromPort": "22",
        "ToPort": "22",
        "CidrIp": "0.0.0.0/0"
      }
    ],
    "SecurityGroupEgress": [
      {
        "IpProtocol": "-1",
        "CidrIp": "0.0.0.0/0"
      }
    ]
  }
}
```

## <a name="3">Webサーバーをセットアップする</a>

_template/cloudformer.template.VPC-intro-step03.json_

### セキュリティグループ
```json
"sgVPCintrosgWEBSG1G90L9V7J9MPB": {
  "Type": "AWS::EC2::SecurityGroup",
  "Properties": {
    "GroupDescription": "WEB-SG",
    "VpcId": {
      "Ref": "vpc3ecd3b5b"
    },
    "SecurityGroupIngress": [
      {
        "IpProtocol": "tcp",
        "FromPort": "22",
        "ToPort": "22",
        "CidrIp": "0.0.0.0/0"
      },
      {
        "IpProtocol": "tcp",
        "FromPort": "80",
        "ToPort": "80",
        "CidrIp": "0.0.0.0/0"
      }
    ],
    "SecurityGroupEgress": [
      {
        "IpProtocol": "-1",
        "CidrIp": "0.0.0.0/0"
      }
    ]
  }
}
},
```

### Webサーバーインストール

接続先を確認

![011](https://farm3.staticflickr.com/2942/15425271672_2fb1131b8c.jpg)

インスタンス作成に使った秘密鍵を手元に用意する

```bash
$ ssh -i my-key.pem ec2-user@54.64.226.210
$ sudo yum -y install httpd
$ sudo service httpd start
$ sudo chkconfig httpd on
```

_http://54.64.226.210/_にアクセスして稼働を確認する。

![012](https://farm6.staticflickr.com/5598/15238934590_b9e47bcbb2.jpg)

## <a name="4">プライベートサブネットを構築する</a>

### ネットワーク図

![Step04](https://farm3.staticflickr.com/2944/15402683306_afbcdc1156.jpg)

_template/cloudformer.template.VPC-intro-step04.json_

### プライベートサブネット

```json
"subnetbbbd4bcc": {
  "Type": "AWS::EC2::Subnet",
  "Properties": {
    "CidrBlock": "10.0.2.0/24",
    "AvailabilityZone": "ap-northeast-1a",
    "VpcId": {
      "Ref": "vpc3ecd3b5b"
    },
    "Tags": [
      {
        "Key": "Name",
        "Value": "プライベートサブネット"
      }
    ]
  }
},
```

### DBサーバーインスタンス

```json
"instancei4a783853": {
  "Type": "AWS::EC2::Instance",
  "Properties": {
    "DisableApiTermination": "FALSE",
    "ImageId": "ami-35072834",
    "InstanceType": "t2.micro",
    "KeyName": "my-key",
    "Monitoring": "false",
    "Tags": [
      {
        "Key": "Name",
        "Value": "DBサーバ"
      }
    ],
    "NetworkInterfaces": [
      {
        "DeleteOnTermination": "true",
        "Description": "Primary network interface",
        "DeviceIndex": 0,
        "SubnetId": {
          "Ref": "subnetbbbd4bcc"
        },
        "PrivateIpAddresses": [
          {
            "PrivateIpAddress": "10.0.2.10",
            "Primary": "true"
          }
        ],
        "GroupSet": [
          {
            "Ref": "sgDBSG"
          }
        ]
      }
    ]
  }
},
```

### セキュリティグループ

```json
"sgDBSG": {
  "Type": "AWS::EC2::SecurityGroup",
  "Properties": {
    "GroupDescription": "WEB-SG",
    "VpcId": {
      "Ref": "vpc3ecd3b5b"
    },
    "SecurityGroupIngress": [
      {
        "IpProtocol": "tcp",
        "FromPort": "3306",
        "ToPort": "3306",
        "CidrIp": "0.0.0.0/0"
      },
      {
        "IpProtocol": "tcp",
        "FromPort": "22",
        "ToPort": "22",
        "CidrIp": "0.0.0.0/0"
      },
      {
        "IpProtocol": "icmp",
        "FromPort": "-1",
        "ToPort": "-1",
        "CidrIp": "0.0.0.0/0"
      }
    ],
    "SecurityGroupEgress": [
      {
        "IpProtocol": "-1",
        "CidrIp": "0.0.0.0/0"
      }
    ]
  }
},
"sgVPCintrosgWEBSG1G90L9V7J9MPB": {
  "Type": "AWS::EC2::SecurityGroup",
  "Properties": {
    "GroupDescription": "DB-SG",
    "VpcId": {
      "Ref": "vpc3ecd3b5b"
    },
    "SecurityGroupIngress": [
      {
        "IpProtocol": "tcp",
        "FromPort": "22",
        "ToPort": "22",
        "CidrIp": "0.0.0.0/0"
      },
      {
        "IpProtocol": "tcp",
        "FromPort": "80",
        "ToPort": "80",
        "CidrIp": "0.0.0.0/0"
      },
      {
        "IpProtocol": "icmp",
        "FromPort": "-1",
        "ToPort": "-1",
        "CidrIp": "0.0.0.0/0"
      }
    ],
    "SecurityGroupEgress": [
      {
        "IpProtocol": "-1",
        "CidrIp": "0.0.0.0/0"
      }
    ]
  }
}
},
```

### Webサーバー経由でDBサーバーにアクセスできるようにする

```bash
$ scp -i my-key.pem my-key.pem ec2-user@54.64.226.223:~
$ ssh -i my-key.pem ec2-user@54.64.226.223
$ ssh -i my-key.pem ec2-user@10.0.2.10
```

## <a name="5">NATサーバーを構築する</a>

### ネットワーク図

![Step05](https://farm3.staticflickr.com/2942/15402967226_791552cd21.jpg)

### NATサーバーの送信元／送信先のチェックを無効にする

![013](https://farm3.staticflickr.com/2944/15426172092_b94348b2e2.jpg)

_template/cloudformer.template.VPC-intro-step05.json_

### プライベートルートテーブル

```json
"rtb1430c371": {
  "Type": "AWS::EC2::RouteTable",
  "Properties": {
    "VpcId": {
      "Ref": "vpc3ecd3b5b"
    },
    "Tags": [
      {
        "Key": "Name",
        "Value": "プライベートルートテーブル"
      }
    ]
  }
},
・・・
"route5": {
  "Type": "AWS::EC2::Route",
  "Properties": {
    "DestinationCidrBlock": "0.0.0.0/0",
    "RouteTableId": {
      "Ref": "rtb1430c371"
    },
    "InstanceId": {
      "Ref": "instanceie680c0ff"
    }
  }
},
```

### NATインスタンス

```json
"instanceie680c0ff": {
  "Type": "AWS::EC2::Instance",
  "Properties": {
    "DisableApiTermination": "FALSE",
    "ImageId": "ami-31c29e30",
    "InstanceType": "t1.micro",
    "KernelId": "aki-176bf516",
    "KeyName": "my-key",
    "Monitoring": "false",
    "Tags": [
      {
        "Key": "Name",
        "Value": "NATサーバ"
      }
    ],
    "NetworkInterfaces": [
      {
        "DeleteOnTermination": "true",
        "Description": "Primary network interface",
        "DeviceIndex": 0,
        "SubnetId": {
          "Ref": "subnetdebf49a9"
        },
        "PrivateIpAddresses": [
          {
            "PrivateIpAddress": "10.0.1.20",
            "Primary": "true"
          }
        ],
        "GroupSet": [
          {
            "Ref": "sgNATSG"
          }
        ],
        "AssociatePublicIpAddress": "true"
      }
    ]
  }
},
```

### セキュリティグループ

```json
"sgNATSG": {
  "Type": "AWS::EC2::SecurityGroup",
  "Properties": {
    "GroupDescription": "NAT Security Group",
    "VpcId": {
      "Ref": "vpc3ecd3b5b"
    },
    "SecurityGroupIngress": [
      {
        "IpProtocol": "tcp",
        "FromPort": "22",
        "ToPort": "22",
        "CidrIp": "10.0.1.10/32"
      },
      {
        "IpProtocol": "tcp",
        "FromPort": "80",
        "ToPort": "80",
        "CidrIp": "0.0.0.0/0"
      },
      {
        "IpProtocol": "tcp",
        "FromPort": "443",
        "ToPort": "443",
        "CidrIp": "0.0.0.0/0"
      }
    ],
    "SecurityGroupEgress": [
      {
        "IpProtocol": "tcp",
        "FromPort": "80",
        "ToPort": "80",
        "CidrIp": "0.0.0.0/0"
      },
      {
        "IpProtocol": "tcp",
        "FromPort": "443",
        "ToPort": "443",
        "CidrIp": "0.0.0.0/0"
      }
    ]
  }
},
```

プライベートルートテーブルの中身

| ディスティネーション（宛先）     | ターゲット    |
|:---------------|:-------------|
| 10.0.0.0/16    |local         |
|  0.0.0.0/0　　　 |NATサーバー     |

## <a name="6">DBを用いたブログシステムの構築</a>

### ネットワーク図

![Step06](https://farm4.staticflickr.com/3935/15422950271_e2ae0b9004.jpg)

### DBセットアップ

```bash
$ ssh -i my-key.pem ec2-user@54.64.226.223
$ ssh -i my-key.pem ec2-user@10.0.2.10
$ sudo yum -y install mysql-server
$ sudo service mysqld start
$ mysqladmin -u root password
New password:
Confirm new password:
$ mysql -u root -p
mysql> CREATE DATABASE wordpress DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;
mysql> grant all on wordpress.* to wordpress@"%" identified by 'wordpresspasswd';
mysql> flush privileges;
mysql> exit;
$ sudo /sbin/chkconfig mysqld on
$ exit
```

### WordPressセットアップ

```bash
$ sudo yum -y install php php-mysql php-mbstring
$ sudo yum -y install mysql
$ mysql -h 10.0.2.10 -u wordpress -pwordpresspasswd
mysql> exit;
$ cd ~
$ wget http://ja.wordpress.org/latest-ja.tar.gz
$ tar xzvf latest-ja.tar.gz
$ cd wordpress/
$ sudo cp -r * /var/www/html/
$ sudo chown apache:apache /var/www/html -R
$ sudo service httpd restart
```

_http://54.64.226.223/_にアクセスする

![014](https://farm4.staticflickr.com/3929/15403461286_b79dd02677.jpg)

![015](https://farm3.staticflickr.com/2944/15239971017_2925806d02.jpg)

![016](https://farm4.staticflickr.com/3933/15239971107_c57129e27f.jpg)

![017](https://farm6.staticflickr.com/5597/15423339061_d893b04a50.jpg)

![018](https://farm4.staticflickr.com/3930/15239971117_3c913f73d3.jpg)

![019](https://farm4.staticflickr.com/3936/15239970927_c0a57b3efa.jpg)

![020](https://farm3.staticflickr.com/2944/15403461306_e156776cf2.jpg)

# 参照

+ [Amazon Virtual Private Cloud](http://docs.aws.amazon.com/ja_jp/AmazonVPC/latest/UserGuide/VPC_Introduction.html)
+ [AWS CloudFormation](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html)
+ [Amazon Web Services 基礎からのネットワーク&サーバー構築 ](http://www.amazon.co.jp/Amazon-Web-Services-%E5%9F%BA%E7%A4%8E%E3%81%8B%E3%82%89%E3%81%AE%E3%83%8D%E3%83%83%E3%83%88%E3%83%AF%E3%83%BC%E3%82%AF-%E3%82%B5%E3%83%BC%E3%83%90%E3%83%BC%E6%A7%8B%E7%AF%89/dp/4822262960)
