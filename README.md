# About
Package stores list of map queryName=>queryPreparedStatement linked to the sqlx/db object.

## Usage example

```go
import (
	"github.com/sda0/go-preparedsqlx"
)

const (
	sqlGetFilesTerms = "sqlGetFilesTerms"
	sqlDeleteFilesTerms = "sqlDeleteFilesTerms"
)
type SQLStorage struct {
	connect   *sqlx.DB
	prepQuery *preparedsqlx.Registry
}

func init() {
	// Add queries to registry
	preparedsqlx.Add(sqlGetFilesTerms, "SELECT * FROM term_file WHERE fid::text=ANY($1)")
	preparedsqlx.Add(sqlDeleteFilesTerms, "DELETE FROM term_file tf WHERE fid=$1")
}

func NewSQLStorage() (*SQLStorage, error) {
	pg := &SQLStorage{}
	_, err := pg.getConnect()
	if err != nil {
		return nil, err
	}
	err = pg.MigrateUp()
	if err != nil {
		pg.connect.Close()
		return nil, err
	}
	// Prepare all registered queries
	pg.prepQuery, err = preparedsqlx.New(pg.connect)
	if err != nil {
		pg.connect.Close()
		return nil, err
	}
	return pg, nil
}

// GetFilesTerms returns all terms associated for requested files
func (s *SQLStorage) GetFilesTerms(ctx context.Context, files []string) ([]*storage.FileTerms, error) {
	getFilesTermsQuery, err := s.prepQuery.Get(sqlGetFilesTerms)
	if err != nil {
		return nil, err
	}
	rows, err := getFilesTermsQuery.QueryxContext(ctx, pg2.Array(files))
	if err != nil {
		return nil, err
	}
	defer rows.Close()
	var result []*storage.FileTerms
	for rows.Next() {
		var r storage.FileTerms
		err = rows.StructScan(&r)
		if err != nil {
			return nil, err
		}
		result = append(result, &r)
	}
	return result, nil
}

// RemoveFilesTerms uses transactions 
func (s *SQLStorage) RemoveFilesTerms(ctx context.Context, vv []*storage.FileTerms) (error) {
	tx, err := s.connect.Beginx()
	if err != nil {
		return nil
	}
	deleteFilesTermsQuery, err := s.prepQuery.GetTx(tx, sqlDeleteFilesTerms)
	if err != nil {
		tx.Rollback()
		return nil
	}
	for _, ft := range vv {
		r, err := deleteFilesTermsQuery.ExecContext(ctx, ft.FileKey, ft.Vocabulary, ft.Term)
		if err != nil {
			tx.Rollback()
			return nil
		}
	}
	return tx.Commit()
}
