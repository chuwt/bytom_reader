# bytom 源码解读

## 1. ./bytomd init 启动
启动的是cmd/bytomd/init.go 下面的initFiles方法，主要是生成配置文件，暂未生成链

## 2. ./bytomd node
启动的是cmd/bytomd/run_node.go 下面的runNode方法，如下
			
			
	func runNode(cmd *cobra.Command, args []string) error {
		// Create & start node
		n := node.NewNode(config)
	
			
其中 n := node.NewNode(config)是核心，根据配置文件生成链,

node.NewNode()方法在 node/node.go 文件中，找到NewNode方法

	func NewNode(config *cfg.Config) *Node {	
		ctx := context.Background()
		if err := lockDataDirectory(config); err != nil {
			cmn.Exit("Error: " + err.Error())
		}
		initLogFile(config)
		initActiveNetParams(config)

前几步主要是设置文件和网络配置

	// Get store
	coreDB := dbm.NewDB("core", config.DBBackend, config.DBDir())
	store := leveldb.NewStore(coreDB)

	tokenDB := dbm.NewDB("accesstoken", config.DBBackend, config.DBDir())
	accessTokens := accesstoken.NewStore(tokenDB)
			
这一步主要是生成accesstoken和core数据库

	// Make event switch
	eventSwitch := types.NewEventSwitch()
	if _, err := eventSwitch.Start(); err != nil {
		cmn.Exit(cmn.Fmt("Failed to start switch: %v", err))
	}
	
这里主要是生成新的时间转换器，继续往下

    txPool := protocol.NewTxPool()
		
生成txPool用于打包交易，NewTxPool()方法在protocol/txpool.go 文件下

	func NewTxPool() *TxPool {
		return &TxPool{
			lastUpdated: time.Now().Unix(),
			pool:        make(map[bc.Hash]*TxDesc),
			utxo:        make(map[bc.Hash]bc.Hash),
			errCache:    lru.New(maxCachedErrTxs),
			newTxCh:     make(chan *types.Tx, maxNewTxChSize),
		}
	}
	
txPool暂不深入,继续往下看

	chain, err := protocol.NewChain(store, txPool)

NewChain创建链，创建链需要传入coredb和txpool，找到protocol.NewChain方法，在protocol/protocol.go 文件下

	
	// NewChain returns a new Chain using store as the underlying storage.
	func NewChain(store Store, txPool *TxPool) (*Chain, error) {
		c := &Chain{
			orphanManage:   NewOrphanManage(),
			txPool:         txPool,
			store:          store,
			processBlockCh: make(chan *processBlockMsg, maxProcessBlockChSize),
		}
		
首先会创建一个Chain对象的引用，在同一文件下来看一下Chain对象的结构体：
	
	type Chain struct {
		index          *state.BlockIndex
		orphanManage   *OrphanManage
		txPool         *TxPool
		store          Store
		processBlockCh chan *processBlockMsg

		cond     sync.Cond
		bestNode *state.BlockNode
	}
	
不太懂OrphanManage，回头理解一下

	c.cond.L = new(sync.Mutex)	

	storeStatus := store.GetStoreStatus()
	if storeStatus == nil {
		if err := c.initChainStatus(); err != nil {
			return nil, err
		}
		storeStatus = store.GetStoreStatus()
	}

	var err error
	if c.index, err = store.LoadBlockIndex(); err != nil {
		return nil, err
	}

	c.bestNode = c.index.GetNode(storeStatus.Hash)
	c.index.SetMainChain(c.bestNode)
	go c.blockProcesser()
	return c, nil
	}
	
这些主要是判断当前区块是否是最新的区块，进行分叉同步，自动切换到最好的链上去，回到node文件里，

	
	ar accounts *account.Manager = nil
	var assets *asset.Registry = nil
	var wallet *w.Wallet = nil
	var txFeed *txfeed.Tracker = nil
	
创建完链之后，分别定义 账户，资产，钱包，交易回馈（？不知道），

先看账户定义account.Manager， 在account/accounts.go

	type Manager struct {
		db     dbm.DB
		chain  *protocol.Chain
		utxoDB *reserver

		cacheMu    sync.Mutex
		cache      *lru.Cache
		aliasCache *lru.Cache

		delayedACPsMu sync.Mutex
		delayedACPs   map[*txbuilder.TemplateBuilder][]*CtrlProgram
	
		accIndexMu sync.Mutex
		accountMu  sync.Mutex
	}

后面会用到，到时候具体分析，继续看node：

	txFeedDB := dbm.NewDB("txfeeds", config.DBBackend, config.DBDir())
	txFeed = txfeed.NewTracker(txFeedDB, chain)
	
生成新的txfeeddb,继续往下看

	if !config.Wallet.Disable {
这里判断是否生成新的钱包，

		walletDB := dbm.NewDB("wallet", config.DBBackend, config.DBDir())

生成wallet数据库
 
		accounts = account.NewManager(walletDB, chain)

生成新用户模块，具体代码如下
		
	func NewManager(walletDB dbm.DB, chain *protocol.Chain) *Manager {
		return &Manager{
			db:          walletDB,
			chain:       chain,
			utxoDB:      newReserver(chain, walletDB),
			cache:       lru.New(maxAccountCache),
			aliasCache:  lru.New(maxAccountCache),
			delayedACPs: make(map[*txbuilder.TemplateBuilder][]*CtrlProgram),
		}
	}

	
	
		assets = asset.NewRegistry(walletDB, chain)
		wallet, err = w.NewWallet(walletDB, accounts, assets, hsm, chain)
		if err != nil {
			log.WithField("error", err).Error("init NewWallet")
		}

		// trigger rescan wallet
		if config.Wallet.Rescan {
			wallet.RescanBlocks()
		}

		// Clean up expired UTXO reservations periodically.
		go accounts.ExpireReservations(ctx, expireReservationsPeriod)
	}


	

	

	


			
			
			
			
			
			
			
						
