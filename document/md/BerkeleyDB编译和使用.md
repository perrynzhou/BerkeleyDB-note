##  BerkeleyDB编译和使用

- 编译BerkeleyDB源代码

  ```
  wget http://download.oracle.com/otn/berkeley-db/db-18.1.40.tar.gz
  tar zxvf db-18.1.40.tar.gz && cd db-18.1.40
  ../dist/configure --enable-debug  CFLAGS="-ggdb3 -O0"  
  make -j4 && make install
  ```

  

- BerkeleyDB开发库

  ```
  Libraries have been installed in:
     /usr/local/BerkeleyDB.18.1/lib
  Header files:
  	/usr/local/BerkeleyDB.18.1/include
  	
  // 把连接库写入到系统ld中	
  echo '/usr/local/BerkeleyDB.18.1/lib' > /etc/ld.so.conf.d/BerkeleyDB.conf
  ```

  

- 编译以及使用

  ```
  // makefile
  all:
  	rm -rf test_sample
  	gcc -g -O0 sample.c  -L /usr/local/BerkeleyDB.18.1/lib -o test_sample -ldb
  
  // 使用例子
  
  // 交互式使用
  [root@CentOS berkeleydb-sample]# ./test_sample -d  /tmp/t1.db  -i y
  127.0.0.1->set ddd
  set command invalid parameters
  127.0.0.1->set hello1 e1            
  set {key=hello1,value=e1} success
  127.0.0.1->get hello1              
  get {key=hello1,value=e1} success
  127.0.0.1->del hello1
  delete {key=hello1}  success.
  127.0.0.1->get hello1
  db_handle->get: BDB0073 DB_NOTFOUND: No matching key/data pair found
  127.0.0.1->quit
  
  // 根据输入执行命令
  
  // set命令
  [root@CentOS berkeleydb-sample]# ./test_sample -d  /tmp/t1.db  -o s -k test1 -v fuck 
  set {key=test1,value=fuck} success
  
  //get命令
  [root@CentOS berkeleydb-sample]# ./test_sample -d  /tmp/t1.db  -o g -k test1      
  get {key=test1,value=fuck} success
  
  // delete命令
  [root@CentOS berkeleydb-sample]# ./test_sample -d  /tmp/t1.db  -o d -k test1   
  delete {key=test1}  success.
  
  
  ```

- 测试代码

  ```
  /*************************************************************************
    > File Name: sample.c
    > Author:perrynzhou 
    > Mail:perrynzhou@gmail.com 
    > Created Time: 日  1/17 09:48:07 2021
   ************************************************************************/
  
  #include <stdio.h>
  #include <string.h>
  #include <stdlib.h>
  #include <unistd.h>
  #include <db.h>
  #define CMD_MAX_BUFFER_LEN (4096)
  enum cmd_type
  {
    SET = 0,
    GET = 1,
    DEL = 2
  };
  typedef struct cmd_t
  {
    int op;
    char *key;
    char *val;
  } cmd;
  static char *db_env_path = "/tmp";
  static void usage(int argc, char *argv[])
  {
    if (argc < 3)
    {
      fprintf(stdout, "usage:%s -d {db-path} -o {p/d/g} -i {y/n} -k {key} -v {value}\n", argv[0]);
      exit(-1);
    }
  }
  inline static void db_set_kv_from_cmd(DBT *k, DBT *v, cmd *c)
  {
    memset(k, 0, sizeof(*k));
    memset(v, 0, sizeof(*v));
    if (c != NULL)
    {
      k->data = c->key;
      k->size = strlen(c->key) + 1;
      if (c->op == SET)
      {
        v->data = c->val;
        v->size = strlen(c->val) + 1;
      }
    }
  }
  static int db_parse_cmd_from_buf(cmd *c, char *buf)
  {
    if (buf == NULL || c == NULL)
    {
      return -1;
    }
    char *p = buf;
    int i = 0;
    char *found = NULL;
    char *argv[3] = {NULL};
    while ((found = strsep(&p, " ")) != NULL)
    {
      argv[i++] = found;
    }
    char *input = argv[0];
    char *key = argv[1];
    char *value = argv[2];
    if (strncmp(input, "set", 3) == 0)
    {
      if (value == NULL)
      {
        return -1;
      }
      c->op = SET;
    }
    else if (strncmp(input, "get", 3) == 0)
    {
      c->op = GET;
    }
    else if (strncmp(input, "del", 3) == 0)
    {
      c->op = DEL;
    }
    if (key != NULL)
    {
      c->key = strdup(key);
    }
    if (value != NULL)
    {
      c->val = strdup(value);
    }
    return 0;
  }
  static int db_parse_cmd_from_param(cmd *c, char *key, char *val, char op)
  {
    if (c == NULL || key == NULL)
    {
      return -1;
    }
    switch (op)
    {
    case 'p':
      c->op = SET;
      break;
    case 'g':
      c->op = GET;
      break;
    case 'd':
      c->op = DEL;
      break;
    default:
      break;
    }
    c->key = strdup(key);
    if (c->op == SET)
    {
      if (val == NULL)
      {
        return -1;
      }
      c->val = val;
    }
    return 0;
  }
  static void cmd_reset(cmd *c)
  {
    if (c->key != NULL)
    {
      free(c->key);
      c->key = NULL;
    }
    if (c->val != NULL)
    {
      free(c->val);
      c->val = NULL;
    }
  }
  static int db_exec_cmd(cmd *c, DB *db_handle)
  {
    if (db_handle == NULL || c == NULL)
    {
      return -1;
    }
  
    DBT k, v;
  
    int ret;
    db_set_kv_from_cmd(&k, &v, c);
    if (c->op == SET)
    {
      ret = db_handle->put(db_handle, NULL, &k, &v, 0);
      if (ret)
      {
        db_handle->err(db_handle, ret, "db_handle->put");
        return -1;
      }
      printf("set {key=%s,value=%s} success\n", (char *)k.data, (char *)v.data);
    }
    else if (c->op == GET)
    {
      ret = db_handle->get(db_handle, NULL, &k, &v, 0);
      if (ret)
      {
        db_handle->err(db_handle, ret, "db_handle->get");
        return -1;
      }
      printf("get {key=%s,value=%s} success\n", (char *)k.data, (char *)v.data);
    }
    else if (c->op == DEL)
    {
      ret = db_handle->del(db_handle, NULL, &k, 0);
      if (ret)
      {
        db_handle->err(db_handle, ret, "db_handle->del");
        return -1;
      }
      printf("delete {key=%s}  success.\n", (char *)k.data);
    }
    return 0;
  }
  static void db_deinit(DB_ENV *env, DB *db_handle)
  {
    if (db_handle != NULL)
    {
      db_handle->close(db_handle, 0);
    }
    if (env != NULL)
    {
      env->close(env, 0);
    }
  }
  static int db_init(DB_ENV **env_ptr, DB **handle_ptr, const char *db_env_path, const char *db_file)
  {
    int ret = db_env_create(env_ptr, 0);
    if (ret != 0)
    {
      fprintf(stderr, "db_env_create failed:: %s\n", db_strerror(ret));
      return -1;
    }
    DB_ENV *env = *env_ptr;
    u_int32_t env_flags = DB_CREATE | DB_INIT_MPOOL;
    ret = env->open(env, db_env_path, env_flags, 0);
    if (ret != 0)
    {
      fprintf(stderr, "onv->open failed: %s", db_strerror(ret));
      exit(1);
    }
    if ((ret = db_create(handle_ptr, env, 0)) != 0)
    {
      fprintf(stderr, "db_create failed: %s\n", db_strerror(ret));
      exit(1);
    }
    DB *db_handle = *handle_ptr;
    if ((ret = db_handle->open(db_handle, NULL, db_file, NULL, DB_BTREE, DB_CREATE, 0664)) != 0)
    {
      db_handle->err(db_handle, ret, "%s", db_file);
      exit(1);
    }
    return 0;
  }
  int main(int argc, char *argv[])
  {
    usage(argc, argv);
    int ch;
    char *db_file = NULL;
    char op_type = '-';
    char *key = NULL;
    char *value = NULL;
    char interactive = 'n';
    while ((ch = getopt(argc, argv, "d:o:i:k:v:")) != EOF)
    {
      switch (ch)
      {
      case 'd': /* Run sample application using a btree database. */
        db_file = strdup(optarg);
        break;
      case 'o': /* Specify the cache size for the environment. */
        op_type = *optarg;
        break;
      case 'k': /* Test on variable-length data. */
        key = strdup(optarg);
        break;
      case 'v': /* Specify the home directory for the environment. */
        value = strdup(optarg);
        break;
      case 'i': /* Specify the home directory for the environment. */
        interactive = *optarg;
        break;
      default:
        break;
      }
    }
    if (op_type == 'p' || op_type == 'd' || op_type == 'g')
    {
      interactive = 'n';
    }
    DB_ENV *env = NULL;
    DB *db_handle = NULL;
    if (db_init(&env, &db_handle, db_env_path, db_file) != 0)
    {
      return -1;
    }
    if (interactive == 'y')
    {
      char buffer[CMD_MAX_BUFFER_LEN] = {'\0'};
      char *base = (char *)&buffer;
      char *buf_ptr = base;
      printf("127.0.0.1->");
      cmd c;
      while (fgets(buf_ptr, CMD_MAX_BUFFER_LEN, stdin) != NULL)
      {
        size_t len = strlen(buf_ptr);
        buf_ptr[--len] = '\0';
        if (len > 0)
        {
          if (strncmp(buf_ptr, "quit", 4) == 0)
          {
            goto quit;
          }
          if (db_parse_cmd_from_buf(&c, buf_ptr) != 0)
          {
            printf("set command invalid parameters\n");
            printf("127.0.0.1->");
            continue;
          }
          db_exec_cmd(&c, db_handle);
          buf_ptr = base;
          cmd_reset(&c);
        }
        printf("127.0.0.1->");
      }
    }
    else
    {
      cmd c;
      if (db_parse_cmd_from_param(&c, key, value, op_type) != 0)
      {
        cmd_reset(&c);
        printf("set command invalid parameters\n");
        goto quit;
      }
      db_exec_cmd(&c, db_handle);
    }
  quit:
    db_deinit(env, db_handle);
    return 0;
  }
  
  ```

  