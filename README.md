<!-- Button to Open the Modal -->
<button class="btn btn-primary" data-bs-toggle="modal" data-bs-target="#simpleFormModal">
  Open Modal
</button>

<!-- Modal Structure -->
<div class="modal fade" id="simpleFormModal" tabindex="-1" aria-labelledby="modalLabel" aria-hidden="true">
  <div class="modal-dialog">
    <div class="modal-content">
      <!-- Modal Header -->
      <div class="modal-header">
        <h5 class="modal-title" id="modalLabel">Simple Form Modal</h5>
        <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close"></button>
      </div>

      <!-- Modal Body (Form) -->
      <div class="modal-body">
        <form [formGroup]="form" (ngSubmit)="onSubmit()">
          <!-- Name Field -->
          <div class="mb-3">
            <label for="name" class="form-label">Name</label>
            <input type="text" id="name" class="form-control" formControlName="name" />
          </div>
        </form>
      </div>

      <!-- Modal Footer -->
      <div class="modal-footer">
        <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Close</button>
        <button type="submit" class="btn btn-primary" [disabled]="form.invalid" (click)="onSubmit()">Submit</button>
      </div>
    </div>
  </div>
</div>
