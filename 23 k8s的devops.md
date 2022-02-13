## DevOps相关学习

### CICD介绍

```
CI/CD是一种通过在应用开发阶段引入自动化工具，来频繁的向客户交付应用的方法,主要针对在集成新代码时所引发的问题,CI、CD是Devops中尤为重要的一个环节。
主要在这三个方面
             持续集成   CI
             持续交付   CD
             持续部署   CD
```

### Pipeline流水线

### 声明式流水线(主流)

在声明式流水线语法中， 流水线过程定义在 Pipeline{}中， Pipeline 块定义了整个流水线中
完成的所有工作，  

```
pipeline {
agent any
stages {
   stage('Build') {
      steps {
//
       }
    }
    stage('Test') {
       steps {
//
       }
    }
    stage('Deploy') {
       steps {
//
        }
    }
  }
}
```

参数说明

- agent any：在任何可用的代理上执行流水线或它的任何阶段，也就是执行流水线过程
  的位置，也可以指定到具体的节点；  
- stage：定义流水线的执行过程（相当于一个阶段），比如上文所示的 Build、Test、Deploy，
  但是这个名字是根据实际情况进行定义的，并非固定的名字；
-  steps：执行某阶段具体的步骤。 

### 脚本式流水线(最好不要用以后可能废弃)

在脚本化流水线语法中，会有一个或多个 Node（节点）块在整个流水线中执行核心工作  

```
node {
    stage('Build') {
//
    }
    stage('Test') {
//
   }
    stage('Deploy') {
//
    }
}
```

参数说明

- node：在任何可用的代理上执行流水线或它的任何阶段， 也可以指定到具体的节点；
- stage： 和声明式的含义一致， 定义流水线的阶段。 Stage 块在脚本化流水线语法中是可
  选的，然而在脚本化流水线中实现 stage 块，可以清楚地在 Jenkins UI 界面中显示每个
  stage 的任务子集。  

**Action**

在声明式流水线中有效的基本语句和表达式遵循与 Groovy 的语法同样的规则，但有以下例
外：
➢ 流水线顶层必须是一个 block，即 pipeline{}；
➢ 分隔符可以不需要分号， 但是每条语句都必须在自己的行上；
➢ 块只能由 Sections、 Directives、 Steps 或 assignment statements 组成；
➢ 属性引用语句被当做是无参数的方法调用，比如 input 会被当做 input()。  

### 流水线中的sections

#### Agent

Agent 表示整个流水线或特定阶段中的步骤和命令执行的位置，该部分必须在 pipeline 块的
顶层被定义， 也可以在 stage 中再次定义， 但是 stage 级别是可选的。  

- any：在任何可用的代理上执行流水线，配置语法  

  ```
  pipeline {
  agent any
  }
  ```

- none： 表示该 Pipeline 脚本没有全局的 agent 配置。 当顶层的 agent 配置为 none 时，每个 stage 部分都需要包含它自己的 agent。配置语法：  

  ```
  pipeline {
       agent none
        stages {
          stage('Stage For Build'){
               agent any
                     }
                }
     }
  ```

- label 选择某个具体的节点执行 Pipeline 命令，例如：agent { label 'my-defined-label' }。
  配置语法：  

  ```
  pipeline {
       agent none
       stages {
          stage('Stage For Build'){
           agent { label 'my-slave-label' }
                    }
             }
         }  
  ```

- node：和 label 配置类似，只不过是可以添加一些额外的配置，比如 customWorkspace；

- dockerfile：使用从源码中包含的 Dockerfile 所构建的容器执行流水线或 stage。 此时对
  应的 agent 写法如下： 

  ```
  agent {
  dockerfile {
        filename 'Dockerfile.build' dir 'build'
        label 'my-defined-label'
        additionalBuildArgs '--build-arg version=1.0.2'
  }
  }
   
  ```

- docker：相当于 dockerfile，可以直接使用 docker 字段指定外部镜像即可，可以省去构建的时间。比如使用 maven 镜像进行打包，同时可以指定args：

  

  ```
  agent{
      docker{
         image 'maven:3-alpine' 
         label 'my-defined-label' 
         args '-v /tmp:/tmp'
         }
      }
  
  ```
  
- Ø kubernetes：Jenkins 也支持使用 Kubernetes 创建 Slave，也就是常说的动态 Slave

  ```
  agent {
      kubernetes { label podlabel yaml """
      kind: Pod 
      metadata:
        name: jenkins-agent 
      spec:
        containers:
        - name: kaniko
          image: gcr.io/kaniko-project/executor:debug 
          imagePullPolicy: Always
          command:
          tty: true 
          volumeMounts:
              - name: aws-secret 
                mountPath: /root/.aws/
              - name: docker-registry-config 
                mountPath: /kaniko/.docker
          restartPolicy: Never 
          volumes:
            - name: aws-secret 
              secret:
                secretName: aws-secret
            - name: docker-registry-config 
              configMap:
                name: docker-registry-config
       """
  }
  ```

  

#### 配置案例

  假设有一个 Java 项目，需要用 mvn 命令进行编译，此时可以使用 maven 的镜像作为 agent。

  ```
  pipeline {
        agent { docker 'maven:3-alpine' }
        stages {
           stage('Example Build') {
               steps {
                   sh 'mvn -B clean verify'
                     }
            }
        } 
}
  ```

  或者另外一种

  ```
在流水线顶层将 agent 定义为 none，那么此时 stage 部分就需要必须包含自己的agent部分
  ```

  ```
  pipeline {
      agent none
      stages {
        stage('Example Build') {
           agent { docker 'maven:3-alpine' }
           steps {
              echo 'Hello, Maven'
              sh 'mvn --version'
           }
         }
         stage('Example Test') {
            agent { docker 'openjdk:8-jre' }
            steps {
              echo 'Hello, JDK'
              sh 'java -version'
               }
          } 
       }
}
  ```

  上述的示例也可以用基于 Kubernetes 的 agent 实现。比如定义具有三个容器的 Pod，分别为jnlp  负责和Jenkins Master 通信）、build（负责执行构建命令）、kubectl  负责执行Kubernetes相关命令），在 steps 中可以通过 containers 字段，选择在某个容器执行

  ```
  pipeline {
     agent {
      kubernetes {
       cloud 'kubernetes-default'
       slaveConnectTimeout 1200
       yaml '''
  apiVersion: v1
  kind: Pod
  spec:
    containers:
      - args: [\'$(JENKINS_SECRET)\', \'$(JENKINS_NAME)\']
        image: 'registry.cn-beijing.aliyuncs.com/citools/jnlp:alpine'
        name: jnlp
        imagePullPolicy: IfNotPresent
      - command:
         - "cat"
        image: "registry.cn-beijing.aliyuncs.com/citools/maven:3.5.3"
        imagePullPolicy: "IfNotPresent"
        name: "build"
        tty: true
      - command:
         - "cat"
        image: "registry.cn-beijing.aliyuncs.com/citools/kubectl:self-1.17"
        imagePullPolicy: "IfNotPresent"
        name: "kubectl"
        tty: true
      '''
  } 
  }
    stages {
      stage('Building') {
        steps {
         container(name: 'build') {
          sh """
          mvm clean install
          """
       }
      }
     }
      stage('Deploy') {
         steps {
           container(name: 'kubectl') {
              sh """
            kubectl get node
               """
           }
        }
      }
     }
}
  ```

  #### POST

  一般用于流水线结束后的进一步处理，比如错误通知等。 Post 可以针对流水线不同的结果做出不同的处理，就像开发程序的错误处理，比如 Python 语言的 try catch。 Post 可以定义在Pipeline 或 stage 中，目前支持以下条件：
  - always：无论 Pipeline 或 stage 的完成状态如何，都允许运行该 post 中定义的指令；
  - changed：只有当前 Pipeline 或 stage 的完成状态与它之前的运行不同时，才允许在该
  post 部分运行该步骤；
  - fixed：当本次 Pipeline 或 stage 成功，且上一次构建是失败或不稳定时，允许运行该
  post 中定义的指令；
  - regression： 当本次 Pipeline 或 stage 的状态为失败、不稳定或终止，且上一次构建的
  状态为成功时，允许运行该 post 中定义的指令；
  - failure：只有当前 Pipeline 或 stage 的完成状态为失败（failure），才允许在 post 部分运
  行该步骤，通常这时在 Web 界面中显示为红色；
  - success：当前状态为成功（success），执行 post 步骤，通常在 Web 界面中显示为蓝色
  或绿色；
  - unstable：当前状态为不稳定（unstable），执行 post 步骤，通常由于测试失败或代码
  违规等造成，在 Web 界面中显示为黄色；
  - aborted：当前状态为终止（aborted），执行该 post 步骤，通常由于流水线被手动终止
  触发，这时在 Web 界面中显示为灰色；
  - unsuccessful：当前状态不是 success 时，执行该 post 步骤；
  - cleanup：无论 pipeline 或 stage 的完成状态如何，都允许运行该 post 中定义的指令。
  和 always 的区别在于， cleanup 会在其它执行之后执行  

  ##### example01

  一般情况下 post 部分放在流水线的底部  

  ```
  pipeline {
      agent any
      stages {
        stage('Example') {
          steps {
            echo 'Hello World'
             }
          }
      }
      post {
        always {
          echo 'I will always say Hello again!'
     }
     }
  }
  ```

  也可以放在stages

  ```
  pipeline {
      agent any
        stages {
          stage('Test') {
            steps {
             sh 'EXECUTE_TEST_COMMAND'
                 }
          post {
            failure {
              echo "Pipeline Testing failure..."
                   }
                 }
           }
       }
  }
  ```

  #### Stages

  Stages 包含一个或多个 stage 指令， 同时可以在 stage 中的 steps 块中定义真正执行的指令。
  比如创建一个流水线， stages 包含一个名为 Example 的 stage，该 stage 执行 echo 'Hello World'
  命令输出 Hello World 字符串：  

  ```
  pipeline {
     agent any
       stages {
         stage('Example') {
           steps {
             echo 'Hello World ${env.BUILD_ID}'
                }
             }
          }
  }
  ```

####   Steps

  Steps 部分在给定的 stage 指令中执行的一个或多个步骤，比如在 steps 定义执行一条 shell 命
  令：  

  ```
  pipeline {
     agent any
       stages {
         stage('Example') {
           steps {
            echo 'Hello World'
              }
           }
       }
  }
  ```

  ### 流水线中的Directives  

  Directives可用于一些执行 stage时的条件判断或预处理一些数据，和 Sections一致，Directives
  不是一个关键字或指令，而是包含了 environment、 options、 parameters、 triggers、 stage、 tools、
  input、 when 等配置。  

  #### Environment 

  主要用于在流水线中配置的一些环境变量，根据配置的位置决定环境变量的作用域。可以定义在 pipeline 中作为全局变量，也可以配置在 stage 中作为该 stage 的环境变量。
  该指令支持一个特殊的方法 credentials()，该方法可用于在 Jenkins 环境中通过标识符访问预
  定义的凭证。对于类型为 Secret Text 的凭证， credentials()可以将该 Secret 中的文本内容赋值给环境变量。 对于类型为标准的账号密码型的凭证，指定的环境变量为 username 和 password，并且也会定义两个额外的环境变量，分别为 MYVARNAME_USR 和 MYVARNAME_PSW。
  假如需要定义个变量名为 CC 的全局变量和一个名为 AN_ACCESS_KEY 的局部变量，并且
  用 credentials 读取一个 Secret 文本，可以通过以下方式定义：  

  ```
  pipeline {
      agent any
       environment { // Pipeline 中定义，属于全局变量
         CC = 'clang'
      }
        stages {
         stage('Example') {
           environment { // 定义在 stage 中，属于局部变量
              AN_ACCESS_KEY = credentials('my-prefined-secret-text')
         }
         steps {
            sh 'printenv'  #输出环境变量
           }
       }
      }
  }
  ```

  #### Options

  Jenkins 流水线支持很多内置指令，比如 retry 可以对失败的步骤进行重复执行 n 次，可以根
  据不同的指令实现不同的效果。 比较常用的指令如下：
  ➢ buildDiscarder ： 保 留 多 少 个 流 水 线 的 构 建 记 录 。 比 如 ： options{ buildDiscarder(logRotator(numToKeepStr: '1')) }；
  ➢ disableConcurrentBuilds：禁止流水线并行执行，防止并行流水线同时访问共享资源导
  致流水线失败。 比如： options { disableConcurrentBuilds() }；
  ➢ disableResume ： 如 果 控 制 器 重 启 ， 禁 止 流 水 线 自 动 恢 复 。 比 如 ： options
  { disableResume() }；
  ➢ newContainerPerStage： agent 为 docker 或 dockerfile 时，每个阶段将在同一个节点的
  新 容 器 中 运 行 ， 而 不 是 所 有 的 阶 段 都 在 同 一 个 容 器 中 运 行 。 比 如 ： options
  { newContainerPerStage () }；
  ➢ quietPeriod：流水线静默期，也就是触发流水线后等待一会在执行。比如： options
  { quietPeriod(30) }；
  ➢ retry：流水线失败后重试次数。比如： options { retry(3) }；
  ➢ timeout：设置流水线的超时时间，超过流水线时间， job 会自动终止。比如： options
  { timeout(time: 1, unit: 'HOURS') }；
  ➢ timestamps：为控制台输出时间戳。比如： options { timestamps() }。
  配置示例如下，只需要添加 options 字段即可  

  ```
  pipeline {
     agent any
     options {
        timeout(time: 1, unit: 'HOURS')
        timestamps()
      }
     stages {
       stage('Example') {
        steps {
          echo 'Hello World'
         }
       }
     }
  }
  ```

  Option 除了写在 Pipeline 顶层，还可以写在 stage 中，但是写在 stage 中的 option 仅支持 retry、
  timeout、 timestamps，或者是和 stage 相关的声明式选项，比如 skipDefaultCheckout。 处于 stage
  级别的 options 写法如下  

  ```
  pipeline {
     agent any
     stages {
      stage('Example') {
        options {
          timeout(time: 1, unit: 'HOURS')
        }
          steps {
             echo 'Hello World'
             }
        }
     }
  }
  ```

  #### Parameters 

  提供了一个用户在触发流水线时应该提供的参数列表，这些用户指定参数的值可以通过 params 对象提供给流水线的 step（步骤）。
  目前支持的参数类型如下：
  ➢ string：字符串类型的参数，例如： parameters { string(name: 'DEPLOY_ENV', defaultValue:'staging', description: '') }，表示定义一个名为 DEPLOY_ENV 的字符型变量，默认值为staging；
  ➢ text：文本型参数，一般用于定义多行文本内容的变量。例如 parameters { text(name:'DEPLOY_TEXT', defaultValue: 'One\nTwo\nThree\n', description: '') }，表示定义一个名  为 DEPLOY_TEXT 的变量，默认值 是'One\nTwo\nThree\n'；
  ➢ booleanParam：布尔型参数，例如: parameters { booleanParam(name: 'DEBUG_BUILD',defaultValue: true, description: '') }；
  ➢ choice：选择型参数，一般用于给定几个可选的值，然后选择其中一个进行赋值，例如：parameters { choice(name: 'CHOICES', choices: ['one', 'two', 'three'], description: '') }，表示定义一个名为 CHOICES 的变量，可选的值为 one、 two、 three；
  ➢ password：密码型变量，一般用于定义敏感型变量，在 Jenkins 控制台会输出为*。例如： parameters { password(name: 'PASSWORD', defaultValue: 'SECRET', description: 'A secret password') }，表示定义一个名为 PASSWORD 的变量，其默认值为 SECRET。  

  ```
  pipeline {
     agent any
     parameters {
       string(name: 'PERSON', defaultValue: 'Mr Jenkins', description:'Who should I say hello to?')
       text(name: 'BIOGRAPHY', defaultValue: '', description: 'Enter some information about the person')
      booleanParam(name: 'TOGGLE', defaultValue: true, description:'Toggle this value')
      choice(name: 'CHOICE', choices: ['One', 'Two', 'Three'],description: 'Pick something')
      password(name: 'PASSWORD', defaultValue: 'SECRET', description:'Enter a password')
  }
    stages {
       stage('Example') {
        steps {
          echo "Hello ${params.PERSON}"
          echo "Biography: ${params.BIOGRAPHY}"
          echo "Toggle: ${params.TOGGLE}"
          echo "Choice: ${params.CHOICE}"
          echo "Password: ${params.PASSWORD}"
              }
         }
       }
  }
  
  ```

  #### Triggers

在 Pipeline 中可以用 triggers 实现自动触发流水线执行任务， 可以通过 Webhook、 Cron、
pollSCM 和 upstream 等方式触发流水线。
假如某个流水线构建的时间比较长，或者某个流水线需要定期在某个时间段执行构建，可以
使用 cron 配置触发器，比如周一到周五每隔四个小时执行一次：  

```
pipeline {
  agent any
  triggers {cron('H */4 * * 1-5')
    }
  stages {
    stage('Example') {
     steps {
          echo 'Hello World'
        }
    }
  }
}
```

注意： H 的意思不是 HOURS 的意思，而是 Hash 的缩写。主要为了解决多个流水线在同一
时间同时运行带来的系统负载压力。  

使用 cron 字段可以定期执行流水线，如果代码更新想要重新触发流水线，可以使用 pollSCM
字段：  

```
pipeline {
    agent any
    triggers {
        pollSCM('H */4 * * 1-5')
        }
	stages {
		stage('Example') {
			steps {
				echo 'Hello World'
				}
			}
		}
}
```

Upstream 可以根据上游 job 的执行结果决定是否触发该流水线。比如当 job1 或 job2 执行成
功时触发该流水线：  

```
pipeline {
    agent any
    triggers {
		upstream(upstreamProjects: 'job1,job2', threshold:hudson.model.Result.SUCCESS)
}
	stages {
        stage('Example') {
			steps {
				echo 'Hello World'
				}
		}
	}
}
```

目前支持的状态有 SUCCESS、 UNSTABLE、 FAILURE、 NOT_BUILT、 ABORTED 等。  

#### Input 

字段可以实现在流水线中进行交互式操作，比如选择要部署的环境、是否继续执行某个阶段等。
配置 Input 支持以下选项：
➢ message： 必选，需要用户进行 input 的提示信息，比如：“是否发布到生产环境？”；
➢ id：可选， input 的标识符，默认为 stage 的名称；
➢ ok： 可选，确认按钮的显示信息，比如：“确定”、“允许”；
➢ submitter： 可选，允许提交 input 操作的用户或组的名称，如果为空，任何登录用户均  可提交 input；
➢ parameters：提供一个参数列表供 input使用

- 假如需要配置一个提示消息为“还继续么”、确认按钮为“继续”、提供一个 PERSON 的变量的参数，并且只能由登录用户为 alice 和 bob 提交的 input 流水线:

```
pipeline {
	agent any
    stages {
		stage('Example') {
		input {
			message "还继续么?"
			ok "继续"
			submitter "alice,bob"
			parameters {
				string(name: 'PERSON', defaultValue: 'Mr Jenkins',description: 'Who should I say hello to?')
			}
            }
			steps {
				echo "Hello, ${PERSON}, nice to meet you."
					}
			}
		}
}
```

#### when

When 指令允许流水线根据给定的条件决定是否应该执行该 stage， when 指令必须包含至少一个条件。如果 when 包含多个条件，所有的子条件必须都返回 True， stage 才能执行。
When 也可以结合 not、 allOf、 anyOf 语法达到更灵活的条件匹配。
目前比较常用的内置条件如下：
➢ branch：当正在构建的分支与给定的分支匹配时，执行这个 stage，例如： when { branch 'master' }。注意， branch 只适用于多分支流水线；
➢ changelog：匹配提交的 changeLog 决定是否构建，例如： when { changelog '.*^\\[DEPENDENCY\\] .+$' }；
➢ environment：当指定的环境变量和给定的变量匹配时，执行这个 stage，例如： when
{ environment name: 'DEPLOY_TO', value: 'production' }；
➢ equals：当期望值和实际值相同时，执行这个 stage，例如： when { equals expected: 2,actual: currentBuild.number }；
➢ expression：当指定的 Groovy 表达式评估为 True，执行这个 stage，例如： when
{ expression { return params.DEBUG_BUILD } }；
➢ tag：如果 TAG_NAME 的值和给定的条件匹配，执行这个 stage，例如： when { tag  "release-*" }；
➢ not：当嵌套条件出现错误时，执行这个 stage，必须包含一个条件，例如： when { not { branch 'master' } }；
➢ allOf：当所有的嵌套条件都正确时， 执行这个 stage，必须包含至少一个条件，例如：
when { allOf { branch 'master'; environment name: 'DEPLOY_TO', value: 'production' } }；
➢ anyOf：当至少有一个嵌套条件为 True 时， 执行这个 stage，例如： when { anyOf { branch  'master'; branch 'staging' } }  

- 当分支为 production 时，执行 Example Deploy 步骤：  

  ```
  pipeline {
      agent any
      stages {
  		stage('Example Build') {
  			steps {
  				echo 'Hello World'
  				}
  			}
  		stage('Example Deploy') {
  			when {
  				branch 'production'
  				}
  			steps {
  				echo 'Deploying'
  			}
          }
  	}
  }
  ```

  也可以同时配置多个条件，比如分支是 production，而且 DEPLOY_TO 变量的值为 production时，才执行 Example Deploy：  

  ```
  pipeline {
      agent any
      stages {
  		stage('Example Build') {
              steps {
  				echo 'Hello World'
  				}
  			}
  		stage('Example Deploy') {
  			when {
  				branch 'production'
  				environment name: 'DEPLOY_TO', value: 'production'
  			}
  			steps {
                  echo 'Deploying'
  			}
  			}
  		}
  }
  ```

  也可以使用 anyOf 进行匹配其中一个条件即可，比如分支为 production， DEPLOY_TO 为
  production 或 staging 时执行 Deploy：  

  ```
  pipeline {
      agent any
      stages {
  		stage('Example Build') {
  			steps {
  				echo 'Hello World'
  				}
  			}
  	stage('Example Deploy') {
  		when {
  		branch 'production'
  		anyOf {
  			environment name: 'DEPLOY_TO', value: 'production'
  			environment name: 'DEPLOY_TO', value: 'staging'
  			}
  	}
  		steps {
  			echo 'Deploying'
  		}
  		}
  	}
  }
  ```

  也可以使用 expression 进行正则匹配，比如当 BRANCH_NAME 为 production 或 staging，并且 DEPLOY_TO 为 production 或 staging 时才会执行 Example Deploy：  

  ```
  pipeline {
  	agent any
  	stages {
  		stage('Example Build') {
  			steps {
  				echo 'Hello World'
  				}
  			}
  		stage('Example Deploy') {
  			when {
  				expression { BRANCH_NAME ==~ /(production|staging)/ }
  				anyOf {
  					environment name: 'DEPLOY_TO', value: 'production'
  					environment name: 'DEPLOY_TO', value: 'staging'
  				}
  			}
  			steps {
  				echo 'Deploying'
                  }
  			}
  }
  }
  ```

  默认情况下，如果定义了某个 stage 的 agent，在进入该 stage 的 agent 后，该 stage 的 when
  条件才会被评估，但是可以通过一些选项更改此选项。比如在进入 stage 的 agent 前评估 when，
  可以使用 beforeAgent，当 when 为 true 时才进行该 stage。
  目前支持的前置条件如下：
  ➢ beforeAgent： 如果 beforeAgent 为 true，则会先评估 when 条件。在 when 条件为 true
  时，才会进入该 stage；
  ➢ beforeInput： 如果 beforeInput 为 true，则会先评估 when 条件。在 when 条件为 true
  时，才会进入到 input 阶段；
  ➢ beforeOptions： 如果 beforeInput 为 true，则会先评估 when 条件。在 when 条件为 true
  时，才会进入到 options 阶段；  

  **ACTION**

  beforeOptions 优先级大于 beforeInput 大于 beforeAgent

- 配置一个 beforeAgent 示例如下：  

  ```
  pipeline {
  agent none
  stages {
  stage('Example Build') {
  steps {
  echo 'Hello World'
  }
  }
  stage('Example Deploy') {
  agent {
  label "some-label"
  }
  when {
  beforeAgent true
  branch 'production'}
  steps {
  echo 'Deploying'
  }
  }
  }
  }
  ```

#### Parallel  

  在声明式流水线中可以使用 Parallel 字段，即可很方便的实现并发构建，比如对分支 A、 B、 C 进行并行处理：  

```
pipeline {
    agent any
    stages {
		stage('Non-Parallel Stage') {
			steps {
				echo 'This stage will be executed first.'
				}
		}
		stage('Parallel Stage') {
			when {
				branch 'master'
			}
			failFast true
			parallel {
				stage('Branch A') {
					agent {
						label "for-branch-a"
					}
					steps {
						echo "On Branch A"
						}
				}
				stage('Branch B') {
					agent {
						label "for-branch-b"
					}
					steps {
							echo "On Branch B"
					}
				}
				stage('Branch C') {
					agent {
						label "for-branch-c"
					}
					stages {
						stage('Nested 1') {
							steps {
									echo "In stage Nested 1 within Branch C"
									}
					}
						stage('Nested 2') {
							steps {
								echo "In stage Nested 2 within Branch C"
								}
						}
					}
				}
				}
			}
		}
}
```

设置 failFast 为 true 表示并行流水线中任意一个 stage 出现错误，其它 stage 也会立即终止。  

执行过程如下

![image-20220213175344438](https://user-images.githubusercontent.com/12670758/153755931-4207fb92-3d01-4d24-9a62-0161a0cc06fb.png)


也可以通过 options 配置在全局：  

```
pipeline {
    agent any
    options {
		parallelsAlwaysFailFast()
		}
	stages {
		stage('Non-Parallel Stage') {
			steps {echo 'This stage will be executed first.'
								}
					}
		stage('Parallel Stage') {
			when {
				branch 'master'
				}
			parallel {
				stage('Branch A') {
					agent {
						label "for-branch-a"
							}
					steps {
						echo "On Branch A"
							}
					}
				stage('Branch B') {
					agent {
						label "for-branch-b"
							}
					steps {
						echo "On Branch B"
							}
					}
				stage('Branch C') {
						agent {
							label "for-branch-c"
								}
						stages {
							stage('Nested 1') {
								steps {
								echo "In stage Nested 1 within Branch C"
									}
					}
							stage('Nested 2') {
								steps {
									echo "In stage Nested 2 within Branch C"
									}
								}
						}
				}
						}
		}
}
}
```

#### if 

if 和when都是用来做判断的，但是if所在的位置和when不同，if需要在stage中定义scrip后，在script中使用

```
pipeline {
    agent any    
    stages {
        stage('flow control') {
            steps {
                script {
                    if ( 10 == 10) {
                        println "pass"
                    }else {
                        println "failed"
                    }
                }
            }
        }
    }
}
```

#### case

可以对参数进行判断，根据判断执行结果

```
pipeline {
    agent any    
    stages {
        stage('Case lab') {
            steps {
                echo 'This stage will be executed first.'
                script{
                    switch("$contraller"){
                    case "Job1":
                        println "This is Job1"
                    break
                    case "Job2":
                        println "This is Job2"
                    break
                    case "Job3":
                        println "This is Job3"
                    break
                    default:
                        echo "############ wrong Job name ############"
                    break
                }
            }
        }
 
    }
}
}
```

### Jenkinfile

创建一个 Jenkinsfile 并将其放置于代码仓库中，有以下好处：
➢ 方便对流水线上的代码进行复查/迭代；
➢ 对管道进行审计跟踪；
➢ 流水线真正的源代码能够被项目的多个成员查看和编辑。  

#### 环境变量

```
1. 静态变量
Jenkins 有许多内置变量可以直接在 Jenkinsfile 中使用， 可以通过 JENKINS_URL/pipelinesyntax/globals#env 获取完整列表。目前比较常用的环境变量如下：
➢ BUILD_ID：当前构建的 ID，与 Jenkins 版本 1.597+中的 BUILD_NUMBER 完全相同；
➢ BUILD_NUMBER：当前构建的 ID，和 BUILD_ID 一致；
➢ BUILD_TAG：用来标识构建的版本号，格式为：jenkins-${JOB_NAME}-${BUILD_NUMBER}，
可以对产物进行命名，比如生产的 jar 包名字、镜像的 TAG 等；
➢ BUILD_URL：本次构建的完整 URL，比如：http://buildserver/jenkins/job/MyJobName/17/；
➢ JOB_NAME：本次构建的项目名称；
➢ NODE_NAME：当前构建节点的名称；
➢ JENKINS_URL： Jenkins 完整的 URL，需要在 System Configuration 设置；
➢ WORKSPACE：执行构建的工作目录。
可以使用 env.BUILD_ID 或 env.JENKINS_URL 引用某个内置变量：
打印输出环境变量 sh "printenv"
```

例如

```
pipeline {
	agent any
	stages {	
		stage('Example') {
			steps {
				echo "Running ${env.BUILD_ID} on ${env.JENKINS_URL}"
				}
				}
			}
		}
```

#### 动态变量

```
pipeline {
  agent any
  environment {
// 使用 returnStdout
    CC = """${sh(
       returnStdout: true,
       script: 'echo "clang"'
      )}"""
// 使用 returnStatus
    EXIT_STATUS = """${sh(
       returnStatus: true,
      script: 'exit 1'
     )}"""
   }
     stages {
       stage('Example') {
         environment {
            DEBUG_FLAGS = '-g'
               }
        steps {
                   sh 'printenv'
           }
      }
   }
}
```

- returnStdout： 将命令的执行结果赋值给变量，比如上述的命令返回的是 clang，此时 CC
  的值为“clang ”。注意后面多了一个空格，可以用.trim()将其删除；
-  returnStatus：将命令的执行状态赋值给变量，比如上述命令的执行状态为 1，此时
  EXIT_STATUS 的值为 1。  

### 凭证管理

Jenkins 的声明式流水线语法有一个 credentials()函数，它支持 secret text （加密文本）、username和 password（用户名和密码）以及 secret file（加密文件） 等。 接下来看一下一些常用的凭证处理
方法。  

#### 加密文本

本实例演示将两个 Secret 文本凭证分配给单独的环境变量来访问 Amazon Web 服务，需要提前创建这两个文件的 credentials（实践的章节会有演示）， Jenkinsfile 文件的内容如下  

```
pipeline {
  agent {
// Define agent details here
    }
    environment {
       AWS_ACCESS_KEY_ID = credentials('jenkins-aws-secret-key-id')
       AWS_SECRET_ACCESS_KEY = credentials('jenkins-aws-secret-access-key')
          }
  stages {
     stage('Example stage 1') {
       steps {
//
       }
   }
     stage('Example stage 2') {
        steps {
//
       }
   }
  }
}
```

上述示例定义了两个全局变量 AWS_ACCESS_KEY_ID 和 AWS_SECRET_ACCESS_KEY，这两个变量引用的是 credentials 的两个加密文本，并且这两个变量均可以在 stages 直接引用（通过$AWS_SECRET_ACCESS_KEY 和$AWS_ACCESS_KEY_ID）。  

#### 用户名密码

```
本示例用来演示 credentials 账号密码的使用，比如使用一个公用账户访问 Bitbucket、GitLab、Harbor 等。 假设已经配置完成了用户名密码形式的 credentials，凭证 ID 为 jenkins-bitbucketcommon-creds。可以用以下方式设置凭证环境变量（BITBUCKET_COMMON_CREDS 名称可以自定义） 
```

```
environment {
BITBUCKET_COMMON_CREDS = credentials('jenkins-bitbucket-commoncreds')
}
```

上述的配置会自动生成 3 个环境变量：
➢ BITBUCKET_COMMON_CREDS： 包含一个以冒号分隔的用户名和密码，格式为
username:password；
➢ BITBUCKET_COMMON_CREDS_USR： 仅包含用户名的附加变量；
➢ BITBUCKET_COMMON_CREDS_PSW： 仅包含密码的附加变量。  

```
pipeline {
  agent any
    environment {
       BITBUCKET_COMMON_CREDS = credentials('jenkins-aws-secret-key-id')
       
          }
  stages {
     stage('Example stage 1') {
       steps {
           echo "$BITBUCKET_COMMON_CREDS"

       }
   }
     stage('Example stage 2') {
        steps {
            // echo ${BITBUCKET_COMMON_CREDS_PSW}
            echo "$BITBUCKET_COMMON_CREDS_USR"

       }
   }
  }
}
```

#### 加密文件

```
需要加密保存的文件，也可以使用 credential，比如链接到 Kubernetes 集群的 kubeconfig 文件等。
假如已经配置好了一个 kubeconfig 文件，此时可以在 Pipeline 中引用该文件：
```

```
pipeline {
    agent {
      // Define agent details here
      }
    environment {
        MY_KUBECONFIG = credentials('my-kubeconfig')
    }
    stages {
      stage('Example stage 1') {
        steps {
          sh("kubectl --kubeconfig $MY_KUBECONFIG get pods")
           }
      }
   }
}
```

### 参数处理

声明式流水线支持很多开箱即用的参数， 可以让流水线接收不同的参数以达到不同的构建
效果，在 Directives 小节讲解的参数均可用在流水线中。
在 Jenkinsfile 中指定的 parameters 会在 Jenkins Web UI 自动生成对应的参数列表，此时可以
在 Jenkins 页面点击 Build With Parameters 来指定参数的值， 这些参数可以通过 params 变量被成
员访问。
假设在 Jenkinsfile 中配置了名为 Greeting 的字符串参数，可以通过${params.Greeting}访问
该参数，比如：  

````
pipeline {
		agent any
		parameters {
			string(name: 'Greeting', defaultValue: 'Hello', description: 'How should I greet the world?')}
			stages {
               stage('Example') {
                 steps {
                   echo "${params.Greeting} World!"
                   }
             }
      }
}
````

#### 使用多个代理

流水线允许在 Jenkins 环境中使用多个代理，这有助于更高级的用例，例如跨多个平台执行构建、测试等。  

主要配置在于agent:xxx

### Shared Libraries

