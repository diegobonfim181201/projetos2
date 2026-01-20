

CREATE DATABASE IF NOT EXISTS digital_bank;
USE digital_bank;

-- ===========================================
-- 1. CONTROLE DE USUÁRIOS INTERNOS
-- ===========================================

CREATE TABLE internal_users (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(150) NOT NULL,
    email VARCHAR(150) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(50),
    status VARCHAR(30),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NULL
);

CREATE TABLE internal_roles (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE internal_permissions (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(150),
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE internal_role_permissions (
    role_id BIGINT,
    permission_id BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (role_id, permission_id),
    FOREIGN KEY (role_id) REFERENCES internal_roles(id),
    FOREIGN KEY (permission_id) REFERENCES internal_permissions(id)
);

CREATE TABLE internal_user_roles (
    user_id BIGINT,
    role_id BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, role_id),
    FOREIGN KEY (user_id) REFERENCES internal_users(id),
    FOREIGN KEY (role_id) REFERENCES internal_roles(id)
);

-- ===========================================
-- 2. CONTAS BANCÁRIAS
-- ===========================================

CREATE TABLE accounts (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_number VARCHAR(30) UNIQUE NOT NULL,
    branch VARCHAR(10),
    type VARCHAR(50),
    status VARCHAR(50),
    currency VARCHAR(10),
    balance DECIMAL(18,2) DEFAULT 0,
    available_balance DECIMAL(18,2) DEFAULT 0,
    overdraft_limit DECIMAL(18,2),
    opened_at TIMESTAMP,
    closed_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE account_settings (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT,
    allow_overdraft BOOLEAN,
    allow_international BOOLEAN,
    allow_pix BOOLEAN,
    allow_withdraw BOOLEAN,
    allow_debit BOOLEAN,
    allow_credit BOOLEAN,
    daily_limit DECIMAL(18,2),
    monthly_limit DECIMAL(18,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id)
);

CREATE TABLE account_status_history (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT,
    old_status VARCHAR(50),
    new_status VARCHAR(50),
    reason TEXT,
    changed_by BIGINT,
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id)
);

-- ===========================================
-- 3. SALDOS E MOVIMENTAÇÕES
-- ===========================================

CREATE TABLE balances (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT,
    currency VARCHAR(10),
    balance DECIMAL(18,2),
    available DECIMAL(18,2),
    blocked DECIMAL(18,2),
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id)
);

CREATE TABLE ledger_entries (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT,
    reference VARCHAR(100),
    type VARCHAR(50),
    direction VARCHAR(10),
    amount DECIMAL(18,2),
    currency VARCHAR(10),
    balance_after DECIMAL(18,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id)
);

CREATE TABLE daily_account_snapshots (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT,
    snapshot_date DATE,
    opening_balance DECIMAL(18,2),
    closing_balance DECIMAL(18,2),
    total_debit DECIMAL(18,2),
    total_credit DECIMAL(18,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id)
);

-- ===========================================
-- 4. CARTÕES
-- ===========================================

CREATE TABLE cards (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT,
    card_number VARCHAR(25),
    last_four_digits VARCHAR(4),
    card_type VARCHAR(50),
    brand VARCHAR(50),
    expiration_date DATE,
    cvv_hash VARCHAR(255),
    status VARCHAR(50),
    is_virtual BOOLEAN,
    issued_at TIMESTAMP,
    activated_at TIMESTAMP NULL,
    blocked_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id)
);

CREATE TABLE card_limits (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    card_id BIGINT,
    daily_limit DECIMAL(18,2),
    monthly_limit DECIMAL(18,2),
    international_limit DECIMAL(18,2),
    online_limit DECIMAL(18,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (card_id) REFERENCES cards(id)
);

CREATE TABLE card_transactions (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    card_id BIGINT,
    merchant_name VARCHAR(200),
    merchant_category VARCHAR(100),
    transaction_type VARCHAR(50),
    amount DECIMAL(18,2),
    currency VARCHAR(10),
    status VARCHAR(50),
    authorization_code VARCHAR(100),
    is_contactless BOOLEAN,
    is_international BOOLEAN,
    transaction_date TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (card_id) REFERENCES cards(id)
);

CREATE TABLE card_security_settings (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    card_id BIGINT,
    allow_online BOOLEAN,
    allow_international BOOLEAN,
    allow_contactless BOOLEAN,
    allow_withdraw BOOLEAN,
    allow_recurring BOOLEAN,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (card_id) REFERENCES cards(id)
);

-- ===========================================
-- 5. FATURAS E PARCELAMENTOS
-- ===========================================

CREATE TABLE credit_invoices (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    card_id BIGINT,
    invoice_month DATE,
    total_amount DECIMAL(18,2),
    minimum_payment DECIMAL(18,2),
    due_date DATE,
    status VARCHAR(50),
    paid_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (card_id) REFERENCES cards(id)
);

CREATE TABLE invoice_items (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    invoice_id BIGINT,
    card_transaction_id BIGINT,
    description VARCHAR(255),
    amount DECIMAL(18,2),
    installment_number INT,
    total_installments INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (invoice_id) REFERENCES credit_invoices(id),
    FOREIGN KEY (card_transaction_id) REFERENCES card_transactions(id)
);

CREATE TABLE installments (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    invoice_item_id BIGINT,
    installment_number INT,
    due_date DATE,
    amount DECIMAL(18,2),
    status VARCHAR(50),
    paid_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (invoice_item_id) REFERENCES invoice_items(id)
);

CREATE TABLE invoice_payments (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    invoice_id BIGINT,
    payment_method VARCHAR(50),
    amount DECIMAL(18,2),
    paid_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (invoice_id) REFERENCES credit_invoices(id)
);

-- ===========================================
-- 6. PIX
-- ===========================================

CREATE TABLE pix_keys (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT,
    key_type VARCHAR(50),
    key_value VARCHAR(200),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id)
);

CREATE TABLE pix_transactions (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    pix_key_id BIGINT,
    account_id BIGINT,
    transaction_type VARCHAR(50),
    direction VARCHAR(10),
    amount DECIMAL(18,2),
    currency VARCHAR(10),
    status VARCHAR(50),
    end_to_end_id VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (pix_key_id) REFERENCES pix_keys(id),
    FOREIGN KEY (account_id) REFERENCES accounts(id)
);

CREATE TABLE pix_refunds (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    pix_transaction_id BIGINT,
    reason TEXT,
    amount DECIMAL(18,2),
    status VARCHAR(50),
    processed_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (pix_transaction_id) REFERENCES pix_transactions(id)
);

CREATE TABLE pix_limits (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT,
    daily_limit DECIMAL(18,2),
    nightly_limit DECIMAL(18,2),
    per_transaction_limit DECIMAL(18,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id)
);

-- ===========================================
-- 7. TRANSFERÊNCIAS
-- ===========================================

CREATE TABLE transfers (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    source_account_id BIGINT,
    destination_account_id BIGINT,
    transfer_type VARCHAR(50),
    amount DECIMAL(18,2),
    currency VARCHAR(10),
    status VARCHAR(50),
    reference VARCHAR(200),
    scheduled_for TIMESTAMP NULL,
    executed_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (source_account_id) REFERENCES accounts(id),
    FOREIGN KEY (destination_account_id) REFERENCES accounts(id)
);

CREATE TABLE transfer_fees (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    transfer_id BIGINT,
    fee_type VARCHAR(50),
    amount DECIMAL(18,2),
    currency VARCHAR(10),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (transfer_id) REFERENCES transfers(id)
);

CREATE TABLE recurring_transfers (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    source_account_id BIGINT,
    destination_account_id BIGINT,
    frequency VARCHAR(50),
    amount DECIMAL(18,2),
    start_date DATE,
    end_date DATE NULL,
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (source_account_id) REFERENCES accounts(id),
    FOREIGN KEY (destination_account_id) REFERENCES accounts(id)
);

-- ===========================================
-- 8. BOLETOS E COBRANÇA
-- ===========================================

CREATE TABLE boletos (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT,
    barcode VARCHAR(255),
    digitable_line VARCHAR(255),
    amount DECIMAL(18,2),
    due_date DATE,
    status VARCHAR(50),
    paid_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id)
);

CREATE TABLE boleto_payments (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    boleto_id BIGINT,
    payment_method VARCHAR(50),
    amount DECIMAL(18,2),
    paid_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (boleto_id) REFERENCES boletos(id)
);

CREATE TABLE chargebacks (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    card_transaction_id BIGINT,
    reason VARCHAR(255),
    status VARCHAR(50),
    amount DECIMAL(18,2),
    opened_at TIMESTAMP,
    closed_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (card_transaction_id) REFERENCES card_transactions(id)
);

-- ===========================================
-- 9. EMPRÉSTIMOS
-- ===========================================

CREATE TABLE loans (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT,
    loan_type VARCHAR(50),
    principal_amount DECIMAL(18,2),
    interest_rate DECIMAL(8,4),
    total_amount DECIMAL(18,2),
    term_months INT,
    status VARCHAR(50),
    approved_at TIMESTAMP NULL,
    disbursed_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id)
);

CREATE TABLE loan_installments (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    loan_id BIGINT,
    installment_number INT,
    due_date DATE,
    principal DECIMAL(18,2),
    interest DECIMAL(18,2),
    total_amount DECIMAL(18,2),
    status VARCHAR(50),
    paid_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (loan_id) REFERENCES loans(id)
);

CREATE TABLE loan_payments (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    loan_installment_id BIGINT,
    payment_method VARCHAR(50),
    amount DECIMAL(18,2),
    paid_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (loan_installment_id) REFERENCES loan_installments(id)
);

CREATE TABLE loan_collaterals (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    loan_id BIGINT,
    collateral_type VARCHAR(50),
    description TEXT,
    estimated_value DECIMAL(18,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (loan_id) REFERENCES loans(id)
);

CREATE TABLE loan_risk_analysis (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    loan_id BIGINT,
    risk_score DECIMAL(6,2),
    probability_default DECIMAL(6,4),
    analyst_comments TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (loan_id) REFERENCES loans(id)
);

-- ===========================================
-- 10. INVESTIMENTOS
-- ===========================================

CREATE TABLE investment_accounts (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT,
    profile VARCHAR(50),
    risk_tolerance VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id)
);

CREATE TABLE investment_products (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    product_name VARCHAR(150),
    product_type VARCHAR(50),
    issuer VARCHAR(150),
    risk_level VARCHAR(50),
    minimum_investment DECIMAL(18,2),
    liquidity_days INT,
    yield_rate DECIMAL(10,4),
    currency VARCHAR(10),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE investment_positions (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    investment_account_id BIGINT,
    product_id BIGINT,
    quantity DECIMAL(18,6),
    average_price DECIMAL(18,6),
    total_invested DECIMAL(18,2),
    current_value DECIMAL(18,2),
    opened_at TIMESTAMP,
    closed_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (investment_account_id) REFERENCES investment_accounts(id),
    FOREIGN KEY (product_id) REFERENCES investment_products(id)
);

CREATE TABLE investment_transactions (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    investment_account_id BIGINT,
    product_id BIGINT,
    transaction_type VARCHAR(50),
    quantity DECIMAL(18,6),
    price DECIMAL(18,6),
    amount DECIMAL(18,2),
    status VARCHAR(50),
    executed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (investment_account_id) REFERENCES investment_accounts(id),
    FOREIGN KEY (product_id) REFERENCES investment_products(id)
);

CREATE TABLE investment_dividends (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    investment_position_id BIGINT,
    dividend_date DATE,
    amount DECIMAL(18,2),
    currency VARCHAR(10),
    credited_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (investment_position_id) REFERENCES investment_positions(id)
);

-- ===========================================
-- 11. CRIPTOMOEDAS
-- ===========================================

CREATE TABLE crypto_wallets (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT,
    wallet_address VARCHAR(255),
    blockchain VARCHAR(100),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id)
);

CREATE TABLE crypto_assets (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    symbol VARCHAR(20),
    name VARCHAR(100),
    blockchain VARCHAR(100),
    decimals INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE crypto_balances (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    crypto_wallet_id BIGINT,
    crypto_asset_id BIGINT,
    balance DECIMAL(36,18),
    available DECIMAL(36,18),
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (crypto_wallet_id) REFERENCES crypto_wallets(id),
    FOREIGN KEY (crypto_asset_id) REFERENCES crypto_assets(id)
);

CREATE TABLE crypto_transactions (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    crypto_wallet_id BIGINT,
    crypto_asset_id BIGINT,
    transaction_type VARCHAR(50),
    amount DECIMAL(36,18),
    fee DECIMAL(36,18),
    tx_hash VARCHAR(255),
    status VARCHAR(50),
    confirmed_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (crypto_wallet_id) REFERENCES crypto_wallets(id),
    FOREIGN KEY (crypto_asset_id) REFERENCES crypto_assets(id)
);

-- ===========================================
-- 12. SEGUROS
-- ===========================================

CREATE TABLE insurance_products (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    product_name VARCHAR(150),
    coverage_type VARCHAR(100),
    premium DECIMAL(18,2),
    deductible DECIMAL(18,2),
    max_coverage DECIMAL(18,2),
    currency VARCHAR(10),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE insurance_policies (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT,
    product_id BIGINT,
    policy_number VARCHAR(100),
    start_date DATE,
    end_date DATE,
    premium DECIMAL(18,2),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id),
    FOREIGN KEY (product_id) REFERENCES insurance_products(id)
);

CREATE TABLE insurance_claims (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    policy_id BIGINT,
    claim_number VARCHAR(100),
    claim_type VARCHAR(100),
    amount_requested DECIMAL(18,2),
    amount_approved DECIMAL(18,2),
    status VARCHAR(50),
    opened_at TIMESTAMP,
    closed_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (policy_id) REFERENCES insurance_policies(id)
);

CREATE TABLE insurance_claim_documents (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    claim_id BIGINT,
    document_type VARCHAR(100),
    document_url VARCHAR(255),
    uploaded_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (claim_id) REFERENCES insurance_claims(id)
);

-- ===========================================
-- 13. CASHBACK E RECOMPENSAS
-- ===========================================

CREATE TABLE cashback_programs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    program_name VARCHAR(150),
    description TEXT,
    cashback_rate DECIMAL(6,4),
    max_cashback DECIMAL(18,2),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE cashback_earnings (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT,
    program_id BIGINT,
    transaction_reference VARCHAR(100),
    amount_earned DECIMAL(18,2),
    currency VARCHAR(10),
    status VARCHAR(50),
    credited_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id),
    FOREIGN KEY (program_id) REFERENCES cashback_programs(id)
);

CREATE TABLE cashback_redemptions (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    cashback_earning_id BIGINT,
    redemption_type VARCHAR(50),
    amount DECIMAL(18,2),
    redeemed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (cashback_earning_id) REFERENCES cashback_earnings(id)
);

-- ===========================================
-- 14. MARKETPLACE
-- ===========================================

CREATE TABLE marketplace_merchants (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    merchant_name VARCHAR(200),
    category VARCHAR(100),
    website VARCHAR(255),
    contact_email VARCHAR(150),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE marketplace_products (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    merchant_id BIGINT,
    product_name VARCHAR(200),
    description TEXT,
    price DECIMAL(18,2),
    currency VARCHAR(10),
    stock_quantity INT,
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (merchant_id) REFERENCES marketplace_merchants(id)
);

CREATE TABLE marketplace_orders (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT,
    merchant_id BIGINT,
    order_number VARCHAR(100),
    total_amount DECIMAL(18,2),
    currency VARCHAR(10),
    status VARCHAR(50),
    ordered_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id),
    FOREIGN KEY (merchant_id) REFERENCES marketplace_merchants(id)
);

CREATE TABLE marketplace_order_items (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    order_id BIGINT,
    product_id BIGINT,
    quantity INT,
    unit_price DECIMAL(18,2),
    total_price DECIMAL(18,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (order_id) REFERENCES marketplace_orders(id),
    FOREIGN KEY (product_id) REFERENCES marketplace_products(id)
);

-- ===========================================
-- 15. OPEN FINANCE / OPEN BANKING
-- ===========================================

CREATE TABLE open_finance_providers (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    provider_name VARCHAR(150),
    api_base_url VARCHAR(255),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE open_finance_consents (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT,
    provider_id BIGINT,
    consent_type VARCHAR(100),
    granted_at TIMESTAMP,
    expires_at TIMESTAMP,
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id),
    FOREIGN KEY (provider_id) REFERENCES open_finance_providers(id)
);

CREATE TABLE open_finance_data_requests (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    consent_id BIGINT,
    endpoint VARCHAR(255),
    request_payload TEXT,
    response_payload TEXT,
    status VARCHAR(50),
    requested_at TIMESTAMP,
    responded_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (consent_id) REFERENCES open_finance_consents(id)
);

-- ===========================================
-- 16. SEGURANÇA
-- ===========================================

CREATE TABLE security_sessions (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT,
    session_token VARCHAR(255),
    ip_address VARCHAR(50),
    device_info TEXT,
    started_at TIMESTAMP,
    ended_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES internal_users(id)
);

CREATE TABLE security_login_attempts (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NULL,
    email VARCHAR(150),
    ip_address VARCHAR(50),
    success BOOLEAN,
    failure_reason VARCHAR(255),
    attempted_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE security_password_resets (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT,
    reset_token VARCHAR(255),
    expires_at TIMESTAMP,
    used_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES internal_users(id)
);

CREATE TABLE security_mfa_devices (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT,
    device_type VARCHAR(50),
    device_identifier VARCHAR(255),
    secret_key VARCHAR(255),
    is_active BOOLEAN,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES internal_users(id)
);

-- ===========================================
-- 17. FRAUDE E ANTIFRAUDE
-- ===========================================

CREATE TABLE fraud_rules (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    rule_name VARCHAR(150),
    description TEXT,
    risk_score INT,
    is_active BOOLEAN,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE fraud_alerts (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT,
    transaction_reference VARCHAR(100),
    rule_id BIGINT,
    risk_score INT,
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id),
    FOREIGN KEY (rule_id) REFERENCES fraud_rules(id)
);

CREATE TABLE fraud_investigations (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    fraud_alert_id BIGINT,
    investigator_id BIGINT,
    findings TEXT,
    resolution VARCHAR(255),
    resolved_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (fraud_alert_id) REFERENCES fraud_alerts(id),
    FOREIGN KEY (investigator_id) REFERENCES internal_users(id)
);

CREATE TABLE fraud_blacklists (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    entity_type VARCHAR(50),
    entity_value VARCHAR(255),
    reason TEXT,
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ===========================================
-- 18. KYC / AML / COMPLIANCE
-- ===========================================

CREATE TABLE kyc_profiles (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT,
    risk_level VARCHAR(50),
    status VARCHAR(50),
    last_reviewed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id)
);

CREATE TABLE kyc_documents (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    kyc_profile_id BIGINT,
    document_type VARCHAR(100),
    document_url VARCHAR(255),
    verification_status VARCHAR(50),
    verified_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (kyc_profile_id) REFERENCES kyc_profiles(id)
);

CREATE TABLE aml_rules (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    rule_name VARCHAR(150),
    description TEXT,
    threshold_amount DECIMAL(18,2),
    is_active BOOLEAN,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE aml_alerts (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT,
    transaction_reference VARCHAR(100),
    rule_id BIGINT,
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id),
    FOREIGN KEY (rule_id) REFERENCES aml_rules(id)
);

CREATE TABLE aml_investigations (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    aml_alert_id BIGINT,
    investigator_id BIGINT,
    findings TEXT,
    resolution VARCHAR(255),
    resolved_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (aml_alert_id) REFERENCES aml_alerts(id),
    FOREIGN KEY (investigator_id) REFERENCES internal_users(id)
);

CREATE TABLE sanctions_lists (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    entity_name VARCHAR(255),
    entity_type VARCHAR(50),
    country VARCHAR(100),
    list_source VARCHAR(150),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ===========================================
-- 19. LGPD / PRIVACIDADE
-- ===========================================

CREATE TABLE privacy_consents (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT,
    consent_type VARCHAR(100),
    granted_at TIMESTAMP,
    revoked_at TIMESTAMP NULL,
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id)
);

CREATE TABLE data_access_requests (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT,
    request_type VARCHAR(100),
    status VARCHAR(50),
    requested_at TIMESTAMP,
    fulfilled_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id)
);

CREATE TABLE data_deletion_requests (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT,
    reason TEXT,
    status VARCHAR(50),
    requested_at TIMESTAMP,
    completed_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id)
);

-- ===========================================
-- 20. LOGS E AUDITORIA
-- ===========================================

CREATE TABLE audit_logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    entity_type VARCHAR(100),
    entity_id BIGINT,
    action VARCHAR(100),
    performed_by BIGINT,
    old_value TEXT,
    new_value TEXT,
    ip_address VARCHAR(50),
    user_agent TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (performed_by) REFERENCES internal_users(id)
);

CREATE TABLE system_logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    log_level VARCHAR(20),
    service_name VARCHAR(100),
    message TEXT,
    stack_trace TEXT,
    occurred_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE event_logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    event_type VARCHAR(100),
    event_source VARCHAR(100),
    payload TEXT,
    occurred_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE api_request_logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    api_key_id BIGINT,
    endpoint VARCHAR(255),
    method VARCHAR(10),
    request_payload TEXT,
    response_payload TEXT,
    status_code INT,
    ip_address VARCHAR(50),
    occurred_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ===========================================
-- 21. RELATÓRIOS E BI
-- ===========================================

CREATE TABLE reports (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    report_name VARCHAR(150),
    description TEXT,
    report_type VARCHAR(50),
    generated_by BIGINT,
    generated_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (generated_by) REFERENCES internal_users(id)
);

CREATE TABLE report_parameters (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    report_id BIGINT,
    parameter_name VARCHAR(100),
    parameter_value VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (report_id) REFERENCES reports(id)
);

CREATE TABLE report_results (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    report_id BIGINT,
    result_data LONGTEXT,
    generated_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (report_id) REFERENCES reports(id)
);

CREATE TABLE dashboards (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    dashboard_name VARCHAR(150),
    owner_id BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (owner_id) REFERENCES internal_users(id)
);

CREATE TABLE dashboard_widgets (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    dashboard_id BIGINT,
    widget_type VARCHAR(100),
    data_source VARCHAR(255),
    configuration JSON,
    position_x INT,
    position_y INT,
    width INT,
    height INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (dashboard_id) REFERENCES dashboards(id)
);

-- ===========================================
-- 22. BACKOFFICE
-- ===========================================

CREATE TABLE backoffice_tasks (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    task_type VARCHAR(100),
    description TEXT,
    status VARCHAR(50),
    assigned_to BIGINT,
    priority VARCHAR(50),
    due_date DATE,
    completed_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (assigned_to) REFERENCES internal_users(id)
);

CREATE TABLE backoffice_comments (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    task_id BIGINT,
    user_id BIGINT,
    comment TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (task_id) REFERENCES backoffice_tasks(id),
    FOREIGN KEY (user_id) REFERENCES internal_users(id)
);

CREATE TABLE backoffice_attachments (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    task_id BIGINT,
    file_name VARCHAR(255),
    file_url VARCHAR(255),
    uploaded_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (task_id) REFERENCES backoffice_tasks(id)
);

-- ===========================================
-- 23. SUPORTE AO CLIENTE
-- ===========================================

CREATE TABLE support_tickets (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT,
    ticket_number VARCHAR(100),
    subject VARCHAR(255),
    description TEXT,
    status VARCHAR(50),
    priority VARCHAR(50),
    opened_at TIMESTAMP,
    closed_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id)
);

CREATE TABLE support_messages (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    ticket_id BIGINT,
    sender_type VARCHAR(50),
    sender_id BIGINT,
    message TEXT,
    sent_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (ticket_id) REFERENCES support_tickets(id)
);

CREATE TABLE support_attachments (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    message_id BIGINT,
    file_name VARCHAR(255),
    file_url VARCHAR(255),
    uploaded_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (message_id) REFERENCES support_messages(id)
);

CREATE TABLE support_sla_policies (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    policy_name VARCHAR(150),
    priority VARCHAR(50),
    response_time_minutes INT,
    resolution_time_minutes INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ===========================================
-- 24. ASSINATURAS E RECORRÊNCIAS
-- ===========================================

CREATE TABLE subscriptions (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT,
    service_name VARCHAR(150),
    provider VARCHAR(150),
    amount DECIMAL(18,2),
    currency VARCHAR(10),
    frequency VARCHAR(50),
    next_billing_date DATE,
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id)
);

CREATE TABLE subscription_payments (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    subscription_id BIGINT,
    payment_date DATE,
    amount DECIMAL(18,2),
    currency VARCHAR(10),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (subscription_id) REFERENCES subscriptions(id)
);

CREATE TABLE subscription_notifications (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    subscription_id BIGINT,
    notification_type VARCHAR(100),
    scheduled_at TIMESTAMP,
    sent_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (subscription_id) REFERENCES subscriptions(id)
);

-- ===========================================
-- 25. PORTABILIDADE
-- ===========================================

CREATE TABLE portability_requests (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT,
    product_type VARCHAR(100),
    source_institution VARCHAR(150),
    destination_institution VARCHAR(150),
    status VARCHAR(50),
    requested_at TIMESTAMP,
    completed_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id)
);

CREATE TABLE portability_documents (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    portability_request_id BIGINT,
    document_type VARCHAR(100),
    document_url VARCHAR(255),
    uploaded_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (portability_request_id) REFERENCES portability_requests(id)
);

-- ===========================================
-- 26. NOTIFICAÇÕES
-- ===========================================

CREATE TABLE notification_channels (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    channel_name VARCHAR(100),
    description TEXT,
    is_active BOOLEAN,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE notifications (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT,
    channel_id BIGINT,
    notification_type VARCHAR(100),
    title VARCHAR(255),
    message TEXT,
    status VARCHAR(50),
    scheduled_at TIMESTAMP,
    sent_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id),
    FOREIGN KEY (channel_id) REFERENCES notification_channels(id)
);

CREATE TABLE notification_templates (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    template_name VARCHAR(150),
    channel_id BIGINT,
    subject VARCHAR(255),
    body TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (channel_id) REFERENCES notification_channels(id)
);

-- ===========================================
-- 27. CONFIGURAÇÕES GLOBAIS
-- ===========================================

CREATE TABLE system_settings (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    setting_key VARCHAR(150) UNIQUE,
    setting_value TEXT,
    description TEXT,
    updated_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE feature_flags (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    feature_name VARCHAR(150),
    is_enabled BOOLEAN,
    environment VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE system_parameters (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    parameter_group VARCHAR(100),
    parameter_key VARCHAR(150),
    parameter_value TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ===========================================
-- 28. APIs E INTEGRAÇÕES
-- ===========================================

CREATE TABLE api_keys (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    client_name VARCHAR(150),
    api_key VARCHAR(255),
    secret_key VARCHAR(255),
    scopes TEXT,
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE api_webhooks (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    api_key_id BIGINT,
    event_type VARCHAR(100),
    callback_url VARCHAR(255),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (api_key_id) REFERENCES api_keys(id)
);

CREATE TABLE api_webhook_logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    webhook_id BIGINT,
    event_payload TEXT,
    response_status INT,
    response_body TEXT,
    delivered_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (webhook_id) REFERENCES api_webhooks(id)
);

-- ===========================================
-- 29. DATA WAREHOUSE / HISTÓRICO
-- ===========================================

CREATE TABLE dw_account_history (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT,
    snapshot_date DATE,
    balance DECIMAL(18,2),
    available_balance DECIMAL(18,2),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE dw_transaction_history (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT,
    transaction_reference VARCHAR(100),
    transaction_type VARCHAR(50),
    direction VARCHAR(10),
    amount DECIMAL(18,2),
    currency VARCHAR(10),
    status VARCHAR(50),
    transaction_date TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE dw_card_spending (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    card_id BIGINT,
    spending_category VARCHAR(100),
    total_amount DECIMAL(18,2),
    period_start DATE,
    period_end DATE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE dw_investment_performance (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    investment_account_id BIGINT,
    product_id BIGINT,
    return_rate DECIMAL(10,4),
    period_start DATE,
    period_end DATE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ===========================================
-- 30. METADADOS E CONTROLE DE MIGRAÇÃO
-- ===========================================

CREATE TABLE schema_versions (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    version VARCHAR(50),
    applied_at TIMESTAMP,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE migration_logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    migration_name VARCHAR(150),
    status VARCHAR(50),
    executed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE environment_configs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    environment VARCHAR(50),
    config_key VARCHAR(150),
    config_value TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ===========================================
-- 31. TABELAS AUXILIARES
-- ===========================================

CREATE TABLE currencies (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    code VARCHAR(10),
    name VARCHAR(100),
    symbol VARCHAR(10),
    decimals INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE countries (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    iso_code VARCHAR(10),
    name VARCHAR(150),
    region VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE states (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    country_id BIGINT,
    name VARCHAR(150),
    abbreviation VARCHAR(10),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (country_id) REFERENCES countries(id)
);

CREATE TABLE cities (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    state_id BIGINT,
    name VARCHAR(150),
    postal_code VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (state_id) REFERENCES states(id)
);

CREATE TABLE languages (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    language_code VARCHAR(10),
    name VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE timezones (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    timezone_name VARCHAR(100),
    utc_offset VARCHAR(10),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ===========================================
-- 32. HISTÓRICO DE CONFIGURAÇÕES
-- ===========================================

CREATE TABLE system_setting_history (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    setting_key VARCHAR(150),
    old_value TEXT,
    new_value TEXT,
    changed_by BIGINT,
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (changed_by) REFERENCES internal_users(id)
);

CREATE TABLE feature_flag_history (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    feature_name VARCHAR(150),
    old_state BOOLEAN,
    new_state BOOLEAN,
    changed_by BIGINT,
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (changed_by) REFERENCES internal_users(id)
);

CREATE TABLE api_key_history (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    api_key_id BIGINT,
    old_status VARCHAR(50),
    new_status VARCHAR(50),
    changed_by BIGINT,
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (api_key_id) REFERENCES api_keys(id),
    FOREIGN KEY (changed_by) REFERENCES internal_users(id)
);

-- ===========================================
-- 33. SIMULAÇÕES E TESTES
-- ===========================================

CREATE TABLE sandbox_accounts (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    sandbox_name VARCHAR(150),
    currency VARCHAR(10),
    initial_balance DECIMAL(18,2),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE sandbox_transactions (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    sandbox_account_id BIGINT,
    transaction_type VARCHAR(50),
    amount DECIMAL(18,2),
    currency VARCHAR(10),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (sandbox_account_id) REFERENCES sandbox_accounts(id)
);

CREATE TABLE sandbox_logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    sandbox_account_id BIGINT,
    log_message TEXT,
    log_level VARCHAR(50),
    occurred_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (sandbox_account_id) REFERENCES sandbox_accounts(id)
);

-- ===========================================
-- 34. TABELAS DE CONTROLE FINANCEIRO INTERNO
-- ===========================================

CREATE TABLE internal_accounts (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_name VARCHAR(150),
    account_type VARCHAR(100),
    currency VARCHAR(10),
    balance DECIMAL(18,2),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE internal_transactions (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    internal_account_id BIGINT,
    transaction_type VARCHAR(50),
    amount DECIMAL(18,2),
    currency VARCHAR(10),
    description TEXT,
    transaction_date TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (internal_account_id) REFERENCES internal_accounts(id)
);

CREATE TABLE internal_cost_centers (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    center_name VARCHAR(150),
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE internal_expenses (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    cost_center_id BIGINT,
    expense_type VARCHAR(100),
    amount DECIMAL(18,2),
    currency VARCHAR(10),
    description TEXT,
    expense_date DATE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (cost_center_id) REFERENCES internal_cost_centers(id)
);

-- ===========================================
-- 35. HISTÓRICO DE COTAÇÕES
-- ===========================================

CREATE TABLE exchange_rates (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    base_currency VARCHAR(10),
    target_currency VARCHAR(10),
    rate DECIMAL(18,8),
    rate_date DATE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE crypto_exchange_rates (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    crypto_symbol VARCHAR(20),
    fiat_currency VARCHAR(10),
    rate DECIMAL(36,18),
    rate_date DATE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE interest_rate_tables (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    rate_type VARCHAR(100),
    rate DECIMAL(10,6),
    effective_from DATE,
    effective_to DATE NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ===========================================
-- 36. HISTÓRICO DE PRODUTOS E OFERTAS
-- ===========================================

CREATE TABLE product_catalog (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    product_name VARCHAR(150),
    product_type VARCHAR(100),
    description TEXT,
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE product_pricing (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    product_id BIGINT,
    price_type VARCHAR(100),
    amount DECIMAL(18,2),
    currency VARCHAR(10),
    effective_from DATE,
    effective_to DATE NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (product_id) REFERENCES product_catalog(id)
);

CREATE TABLE product_features (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    product_id BIGINT,
    feature_name VARCHAR(150),
    feature_description TEXT,
    is_active BOOLEAN,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (product_id) REFERENCES product_catalog(id)
);

CREATE TABLE product_bundles (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    bundle_name VARCHAR(150),
    description TEXT,
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE product_bundle_items (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    bundle_id BIGINT,
    product_id BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (bundle_id) REFERENCES product_bundles(id),
    FOREIGN KEY (product_id) REFERENCES product_catalog(id)
);

-- ===========================================
-- 37. GOVERNANÇA E POLÍTICAS
-- ===========================================

CREATE TABLE governance_policies (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    policy_name VARCHAR(150),
    policy_type VARCHAR(100),
    description TEXT,
    effective_from DATE,
    effective_to DATE NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE governance_policy_versions (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    policy_id BIGINT,
    version VARCHAR(50),
    content TEXT,
    published_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (policy_id) REFERENCES governance_policies(id)
);

CREATE TABLE governance_policy_acceptance (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    policy_version_id BIGINT,
    user_id BIGINT,
    accepted_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (policy_version_id) REFERENCES governance_policy_versions(id),
    FOREIGN KEY (user_id) REFERENCES internal_users(id)
);

-- ===========================================
-- 38. CONTROLE DE DISPOSITIVOS
-- ===========================================

CREATE TABLE registered_devices (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT,
    device_type VARCHAR(100),
    device_identifier VARCHAR(255),
    os_version VARCHAR(100),
    app_version VARCHAR(50),
    last_used_at TIMESTAMP,
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id)
);

CREATE TABLE device_sessions (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    device_id BIGINT,
    session_token VARCHAR(255),
    started_at TIMESTAMP,
    ended_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (device_id) REFERENCES registered_devices(id)
);

CREATE TABLE device_risk_scores (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    device_id BIGINT,
    risk_score DECIMAL(6,2),
    assessed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (device_id) REFERENCES registered_devices(id)
);

-- ===========================================
-- 39. MOTOR DE REGRAS
-- ===========================================

CREATE TABLE rule_engines (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    engine_name VARCHAR(150),
    description TEXT,
    version VARCHAR(50),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE business_rules (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    rule_engine_id BIGINT,
    rule_name VARCHAR(150),
    rule_expression TEXT,
    action_type VARCHAR(100),
    priority INT,
    is_active BOOLEAN,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (rule_engine_id) REFERENCES rule_engines(id)
);

CREATE TABLE rule_execution_logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    rule_id BIGINT,
    execution_context TEXT,
    result VARCHAR(100),
    executed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (rule_id) REFERENCES business_rules(id)
);

-- ===========================================
-- 40. HISTÓRICO DE ERROS E EXCEÇÕES
-- ===========================================

CREATE TABLE error_codes (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    error_code VARCHAR(50),
    error_message TEXT,
    severity VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE error_logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    error_code_id BIGINT,
    service_name VARCHAR(100),
    error_details TEXT,
    stack_trace TEXT,
    occurred_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (error_code_id) REFERENCES error_codes(id)
);

CREATE TABLE exception_handling_policies (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    policy_name VARCHAR(150),
    description TEXT,
    retry_strategy VARCHAR(100),
    max_retries INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ===========================================
-- 41. CACHE E PERFORMANCE
-- ===========================================

CREATE TABLE cache_entries (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    cache_key VARCHAR(255),
    cache_value LONGTEXT,
    expires_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE cache_invalidation_logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    cache_key VARCHAR(255),
    invalidated_by VARCHAR(100),
    invalidated_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE performance_metrics (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    service_name VARCHAR(100),
    metric_name VARCHAR(150),
    metric_value DECIMAL(18,6),
    recorded_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE performance_alerts (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    metric_id BIGINT,
    threshold_value DECIMAL(18,6),
    alert_message TEXT,
    triggered_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (metric_id) REFERENCES performance_metrics(id)
);

-- ===========================================
-- 42. VERSIONAMENTO DE API
-- ===========================================

CREATE TABLE api_versions (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    api_name VARCHAR(150),
    version VARCHAR(50),
    status VARCHAR(50),
    released_at DATE,
    deprecated_at DATE NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE api_endpoints (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    api_version_id BIGINT,
    endpoint_path VARCHAR(255),
    method VARCHAR(10),
    description TEXT,
    is_public BOOLEAN,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (api_version_id) REFERENCES api_versions(id)
);

CREATE TABLE api_endpoint_permissions (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    api_endpoint_id BIGINT,
    role_id BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (api_endpoint_id) REFERENCES api_endpoints(id),
    FOREIGN KEY (role_id) REFERENCES internal_roles(id)
);

-- ===========================================
-- 43. WEBHOOKS INTERNOS
-- ===========================================

CREATE TABLE internal_webhooks (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    event_type VARCHAR(100),
    callback_url VARCHAR(255),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE internal_webhook_logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    webhook_id BIGINT,
    event_payload TEXT,
    response_status INT,
    response_body TEXT,
    delivered_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (webhook_id) REFERENCES internal_webhooks(id)
);

-- ===========================================
-- 44. ORQUESTRAÇÃO DE SERVIÇOS
-- ===========================================

CREATE TABLE service_registry (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    service_name VARCHAR(150),
    service_url VARCHAR(255),
    status VARCHAR(50),
    last_heartbeat TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE service_dependencies (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    service_id BIGINT,
    dependency_service_id BIGINT,
    dependency_type VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (service_id) REFERENCES service_registry(id),
    FOREIGN KEY (dependency_service_id) REFERENCES service_registry(id)
);

CREATE TABLE service_health_checks (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    service_id BIGINT,
    status VARCHAR(50),
    response_time_ms INT,
    checked_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (service_id) REFERENCES service_registry(id)
);

-- ===========================================
-- 45. FILAS E PROCESSAMENTO ASSÍNCRONO
-- ===========================================

CREATE TABLE message_queues (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    queue_name VARCHAR(150),
    description TEXT,
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE queue_messages (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    queue_id BIGINT,
    message_payload LONGTEXT,
    status VARCHAR(50),
    attempts INT,
    next_retry_at TIMESTAMP NULL,
    processed_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (queue_id) REFERENCES message_queues(id)
);

CREATE TABLE dead_letter_queues (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    original_queue_id BIGINT,
    message_payload LONGTEXT,
    failure_reason TEXT,
    failed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (original_queue_id) REFERENCES message_queues(id)
);

-- ===========================================
-- 46. JOBS E AGENDAMENTOS
-- ===========================================

CREATE TABLE scheduled_jobs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    job_name VARCHAR(150),
    job_type VARCHAR(100),
    schedule_expression VARCHAR(100),
    status VARCHAR(50),
    last_run_at TIMESTAMP NULL,
    next_run_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE job_execution_logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    job_id BIGINT,
    status VARCHAR(50),
    started_at TIMESTAMP,
    ended_at TIMESTAMP NULL,
    output TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (job_id) REFERENCES scheduled_jobs(id)
);

CREATE TABLE job_parameters (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    job_id BIGINT,
    parameter_name VARCHAR(100),
    parameter_value VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (job_id) REFERENCES scheduled_jobs(id)
);

-- ===========================================
-- 47. ARQUIVOS E STORAGE
-- ===========================================

CREATE TABLE file_storage (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    file_name VARCHAR(255),
    file_type VARCHAR(100),
    file_size BIGINT,
    storage_location VARCHAR(255),
    checksum VARCHAR(255),
    uploaded_by BIGINT,
    uploaded_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (uploaded_by) REFERENCES internal_users(id)
);

CREATE TABLE file_access_logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    file_id BIGINT,
    accessed_by BIGINT,
    access_type VARCHAR(50),
    accessed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (file_id) REFERENCES file_storage(id),
    FOREIGN KEY (accessed_by) REFERENCES internal_users(id)
);

CREATE TABLE file_retention_policies (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    file_type VARCHAR(100),
    retention_days INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ===========================================
-- 48. MONITORAMENTO E OBSERVABILIDADE
-- ===========================================

CREATE TABLE monitoring_services (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    service_name VARCHAR(150),
    environment VARCHAR(50),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE monitoring_metrics (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    monitoring_service_id BIGINT,
    metric_name VARCHAR(150),
    metric_value DECIMAL(18,6),
    recorded_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (monitoring_service_id) REFERENCES monitoring_services(id)
);

CREATE TABLE monitoring_alerts (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    monitoring_metric_id BIGINT,
    threshold_value DECIMAL(18,6),
    alert_message TEXT,
    status VARCHAR(50),
    triggered_at TIMESTAMP,
    resolved_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (monitoring_metric_id) REFERENCES monitoring_metrics(id)
);

-- ===========================================
-- 49. MODELOS DE DADOS AUXILIARES
-- ===========================================

CREATE TABLE currencies_supported (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    currency_code VARCHAR(10),
    is_active BOOLEAN,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE countries_supported (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    country_code VARCHAR(10),
    is_active BOOLEAN,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE payment_methods (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    method_name VARCHAR(100),
    description TEXT,
    is_active BOOLEAN,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE payment_channels (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    channel_name VARCHAR(100),
    description TEXT,
    is_active BOOLEAN,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE transaction_categories (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    category_name VARCHAR(150),
    parent_category_id BIGINT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (parent_category_id) REFERENCES transaction_categories(id)
);

-- ===========================================
-- 50. FINALIZAÇÃO
-- ===========================================

SET FOREIGN_KEY_CHECKS = 1;

