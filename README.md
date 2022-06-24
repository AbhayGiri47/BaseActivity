# BaseActivity
create baseactivity of project



abstract class BaseActivity<in T : ViewDataBinding> : RuntimePermissionsActivity() {

    private var progressDialog: ProgressBar? = null
    private var v: View? = null

    private lateinit var mBinding: T

    @Inject
    lateinit var sharedPrefsHelper: SharedPrefsHelper
    private var permissionAllowed = false

    override fun onCreate(savedInstanceState: Bundle?) {
        AndroidInjection.inject(this)
        super.onCreate(savedInstanceState)
        mBinding = DataBindingUtil.setContentView(this, contentView())
        initUI(mBinding)
        enableHome()?.apply {
            visibility = View.VISIBLE
            setOnClickListener {
                startActivity(
                    Intent(
                        this@BaseActivity,
                        DashboardActivity::class.java
                    ).apply {
                        flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK
                    })
            }
        }

        val encodedKey = sharedPrefsHelper[Constant.SECRET_KEY, ""]
        if (null == encodedKey || encodedKey.isEmpty()) {
            val secureRandom = SecureRandom()
            var keyGenerator: KeyGenerator? = null
            try {
                keyGenerator = KeyGenerator.getInstance(Constant.KEY_SPEC_ALGORITHM)
            } catch (e: NoSuchAlgorithmException) {
//                    e.printStackTrace()
            }

            keyGenerator!!.init(Constant.OUTPUT_KEY_LENGTH, secureRandom)
            val secretKey = keyGenerator.generateKey()
            saveSecretKey(secretKey)
        }

    }

    open fun getSecretKey(): SecretKeySpec {
        val decodedKey = Base64.decode(sharedPrefsHelper[Constant.SECRET_KEY, ""], Base64.NO_WRAP)
        return SecretKeySpec(decodedKey, 0, decodedKey.size, Constant.KEY_SPEC_ALGORITHM)
    }

    fun saveSecretKey(secretKey: SecretKey) {
        val encodedKey = Base64.encodeToString(secretKey.encoded, Base64.NO_WRAP)
        sharedPrefsHelper[Constant.SECRET_KEY] = encodedKey
    }

    abstract fun contentView(): Int
    abstract fun initUI(binding: T)
    abstract fun enableHome(): ImageView?

    fun showToast(message: String) {
        val toast = Toast.makeText(this, message, Toast.LENGTH_SHORT)
        toast.setGravity(Gravity.CENTER, 0, 0)
        toast.show()
    }

    protected open fun fragmentTransaction(
        transactionType: Int,
        fragment: Fragment,
        container: Int,
        isAddToBackStack: Boolean,
        bundle: Bundle?
    ) {
        if (bundle != null) {
            fragment.arguments = bundle
        }

        val trans = supportFragmentManager.beginTransaction()
        when (transactionType) {
            ADD_FRAGMENT -> trans.add(container, fragment, fragment.javaClass.simpleName)
            REPLACE_FRAGMENT -> {
                trans.replace(container, fragment, fragment.javaClass.simpleName)
                if (isAddToBackStack) trans.addToBackStack(null)
            }
        }
        trans.commit()
    }

    companion object {
        const val ADD_FRAGMENT = 0
        const val REPLACE_FRAGMENT = 1
        const val App_Version = "Version " + BuildConfig.VERSION_NAME
    }


}



RuntimePermissionsActivity.kt


abstract class RuntimePermissionsActivity : AppCompatActivity() {
    private var mErrorString: SparseIntArray? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        mErrorString = SparseIntArray()
    }

    override fun onRequestPermissionsResult(requestCode: Int, permissions: Array<String>, grantResults: IntArray) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults)
        var permissionCheck = PackageManager.PERMISSION_GRANTED
        for (permission in grantResults) {
            permissionCheck = permissionCheck + permission
        }
        if (grantResults.size > 0 && permissionCheck == PackageManager.PERMISSION_GRANTED) {
            onPermissionsGranted(requestCode)
        } else {
            Snackbar.make(findViewById(android.R.id.content), mErrorString!!.get(requestCode),
                    Snackbar.LENGTH_INDEFINITE).setAction("ENABLE"
            ) {
                val intent = Intent()
                intent.action = Settings.ACTION_APPLICATION_DETAILS_SETTINGS
                intent.addCategory(Intent.CATEGORY_DEFAULT)
                intent.data = Uri.parse("package:$packageName")
                intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
                intent.addFlags(Intent.FLAG_ACTIVITY_NO_HISTORY)
                intent.addFlags(Intent.FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS)
                startActivity(intent)
            }.show()
        }
    }

    fun requestAppPermissions(requestedPermissions: Array<String>,
                              stringId: Int, requestCode: Int) {
        mErrorString!!.put(requestCode, stringId)
        var permissionCheck = PackageManager.PERMISSION_GRANTED
        var shouldShowRequestPermissionRationale = false
        for (permission in requestedPermissions) {
            permissionCheck = permissionCheck + ContextCompat.checkSelfPermission(this, permission)
            shouldShowRequestPermissionRationale = shouldShowRequestPermissionRationale || ActivityCompat.shouldShowRequestPermissionRationale(this, permission)
        }
        if (permissionCheck != PackageManager.PERMISSION_GRANTED) {
            if (shouldShowRequestPermissionRationale) {
                Snackbar.make(findViewById(android.R.id.content), stringId,
                        Snackbar.LENGTH_INDEFINITE).setAction("GRANT"
                )
                {
                    ActivityCompat.requestPermissions(this@RuntimePermissionsActivity, requestedPermissions, requestCode)
                }.show()

            } else {
                ActivityCompat.requestPermissions(this, requestedPermissions, requestCode)
            }
        } else {
            onPermissionsGranted(requestCode)
        }
    }

    abstract fun onPermissionsGranted(requestCode: Int)
}
